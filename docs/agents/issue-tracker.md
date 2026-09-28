# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues. Use the `gh` CLI for all operations.

## Conventions

- **Create an issue**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --comments`, filtering comments by `jq` and also fetching labels.
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

Infer the repo from `git remote -v`; `gh` does this automatically when run inside a clone.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same labels and states as issues, using the `gh pr` equivalents:

- **Read a PR**: `gh pr view <number> --comments` and `gh pr diff <number>` for the diff.
- **List external PRs for triage**: `gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments` then keep only `authorAssociation` of `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE` (drop `OWNER`/`MEMBER`/`COLLABORATOR`).
- **Comment / label / close**: `gh pr comment`, `gh pr edit --add-label`/`--remove-label`, `gh pr close`.

GitHub shares one number space across issues and PRs, so a bare `#42` may be either: resolve with `gh pr view 42` and fall back to `gh issue view 42`.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **child** issues as tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog body. `gh issue create --label wayfinder:map`.
- **Child ticket**: an issue linked to the map as a GitHub sub-issue (`gh api` on the sub-issues endpoint). Where sub-issues aren't enabled, add the child to a task list in the map body and put `Part of #<map>` at the top of the child body. Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: GitHub's **native issue dependencies**, the canonical, UI-visible representation. Add an edge with `gh api --method POST repos/<owner>/<repo>/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`, where `<blocker-db-id>` is the blocker's numeric **database id** (`gh api repos/<owner>/<repo>/issues/<n> --jq .id`, _not_ the `#number` or `node_id`). GitHub reports `issue_dependencies_summary.blocked_by` for open blockers; it is eventually consistent after dependency changes. Where dependencies aren't available, fall back to a `Blocked by: #<n>, #<n>` line at the top of the child body. A ticket is unblocked when every blocker is closed.
- **Frontier query**: list the map's open children, drop any with an open blocker or an assignee, first in map order wins. Use the fail-closed procedure in [Frontier query: fail closed on uncertain dependencies](#frontier-query-fail-closed-on-uncertain-dependencies). Do not rely on the summary or raw blocker-list length alone.
- **Claim**: `gh issue edit <n> --add-assignee @me`, the session's first write.
- **Resolve**: `gh issue comment <n> --body "<answer>"`, then `gh issue close <n>`, then append a context pointer (gist + link) to the map's Decisions-so-far.

## Verified behaviour on this repo

Confirmed by probe on 2026-09-28: this repo has **native sub-issues and native issue dependencies both enabled**, so the body-convention fallback above is not needed.

- `POST repos/<o>/<r>/issues/<n>/sub_issues -F sub_issue_id=<db-id>` links a child; the parent then reports `sub_issues_summary.total`, and the child reports `parent_issue_url`. The summary may lag: during map charting, the listing contained all 10 linked children while the summary briefly reported 9. Use `GET .../sub_issues` as the membership source of truth.
- `POST repos/<o>/<r>/issues/<n>/dependencies/blocked_by -F issue_id=<db-id>` adds a blocking edge. The `issue_dependencies_summary.blocked_by` summary counts open blockers once its update propagates; it may briefly be stale after changes.
- **Gotcha — dependency reads are eventually consistent.** In probes on this repo, the summary was stale immediately after adding an edge; after closing a blocker, the summary and raw listing could also briefly retain the blocker. The raw listing includes each dependency's current `state`, which allows closed issues to be filtered out. Use both reads with the conservative rule below; probe issues were deleted after testing.

## Frontier query: fail closed on uncertain dependencies

A frontier ticket must be open, have no **open** blockers, and be unassigned. GitHub's dependency reads are eventually consistent: the issue summary can lag after adding an edge or closing a blocker, while the dependency listing can retain a closed issue. A raw listing length is wrong because it counts closed dependencies too. GitHub's [List dependencies an issue is blocked by](https://docs.github.com/en/rest/issues/issue-dependencies#list-dependencies-an-issue-is-blocked-by) response includes each blocking issue's `state`.

To avoid duplicate claims, treat a ticket as blocked if either read reports an open blocker. Filter the listing by the blocker's `state`, and combine it with the summary:

```bash
set -euo pipefail
R=kchapl/greentech; MAP=<map-number>
for n in $(gh api repos/$R/issues/$MAP/sub_issues --jq '.[] | select(.state=="open") | .number'); do
  summary=$(gh api repos/$R/issues/$n --jq '.issue_dependencies_summary.blocked_by')
  listing=$(gh api repos/$R/issues/$n/dependencies/blocked_by \
    --jq '[.[] | select(.state=="open")] | length')
  assignee=$(gh api repos/$R/issues/$n --jq '.assignee != null')

  { [ "$summary" -gt 0 ] || [ "$listing" -gt 0 ]; } && continue # blocked; fail closed
  [ "$assignee" = "true" ] && continue                         # already claimed
  echo "FRONTIER: #$n"; break
 done
```

The listing filter matters: `GET .../dependencies/blocked_by` includes closed blocker issues. On this repo, after closing a blocker the unfiltered list still contained it, but filtering entries to `state=="open"` returned zero in all 3 close probes. The summary remained stale in some immediate reads, so the conservative OR may temporarily defer a now-unblocked ticket. Re-read before concluding that no work is available.

**Re-verify at claim time.** Repeat both dependency reads immediately before `gh issue edit <n> --add-assignee @me`. If either reports an open blocker, skip the ticket; falsely skipping delays work, while falsely claiming blocked work risks conflicting sessions. If either API read fails, stop the query rather than treating the missing result as zero. These reads are a conservative operational check, not an atomic lock; the assignment is still the claim.
