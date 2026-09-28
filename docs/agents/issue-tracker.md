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
- **Blocking**: GitHub's **native issue dependencies**, the canonical, UI-visible representation. Add an edge with `gh api --method POST repos/<owner>/<repo>/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`, where `<blocker-db-id>` is the blocker's numeric **database id** (`gh api repos/<owner>/<repo>/issues/<n> --jq .id`, _not_ the `#number` or `node_id`). GitHub reports `issue_dependencies_summary.blocked_by` (open blockers only, the live gate). Where dependencies aren't available, fall back to a `Blocked by: #<n>, #<n>` line at the top of the child body. A ticket is unblocked when every blocker is closed.
- **Frontier query**: list the map's open children, drop any with an open blocker or an assignee, first in map order wins. **Do not read `issue_dependencies_summary` alone** — see [Frontier query: read both endpoints](#frontier-query-read-both-endpoints) below for the fail-safe form.
- **Claim**: `gh issue edit <n> --add-assignee @me`, the session's first write.
- **Resolve**: `gh issue comment <n> --body "<answer>"`, then `gh issue close <n>`, then append a context pointer (gist + link) to the map's Decisions-so-far.

## Verified behaviour on this repo

Confirmed by probe on 2026-09-28: this repo has **native sub-issues and native issue dependencies both enabled**, so the body-convention fallback above is not needed.

- `POST repos/<o>/<r>/issues/<n>/sub_issues -F sub_issue_id=<db-id>` links a child; the parent then reports `sub_issues_summary.total`, and the child reports `parent_issue_url`.
- `POST repos/<o>/<r>/issues/<n>/dependencies/blocked_by -F issue_id=<db-id>` adds a blocking edge; `issue_dependencies_summary.blocked_by` counts **open** blockers only, so it drops to 0 the moment the blocker closes (`total_blocked_by` stays as history). That is exactly the frontier gate.
- **Gotcha — dependency reads are eventually consistent, and stale in _opposite_ directions.** Measured on this repo over 3 trials per direction (probe issues created, wired, then deleted):

  | Read immediately after… | `issue_dependencies_summary.blocked_by` | `GET .../dependencies/blocked_by` |
  | ----------------------- | -------------------------------------- | ---------------------------------- |
  | **adding** a blocking edge   | stale `0`, correct after ~2s (3/3) | **correct immediately** (3/3)  |
  | **closing** a blocker        | stale `1`, correct after ~2s (3/3) | **still stale at ~2s** (3/3)    |

  Neither endpoint is safe on its own in both directions, so the frontier query must combine them — see below.

## Frontier query: read both endpoints

A frontier query runs twice: once after wiring blocking edges (the *add* path), and once after a blocker is closed (the *unblock* path). Each has a different stale endpoint, so a single read is unsafe in one direction or the other.

The asymmetry sets which way to fail. Reporting a ticket as **blocked** when it is not is harmless: the session picks another, and the ticket is found next session. Reporting it as **unblocked** when it is blocked causes two sessions to claim the same ticket. So treat a ticket as blocked if **either** endpoint reports an open blocker:

```bash
MAP=<map-number>; R=kchapl/greentech

# children of the map, in map order, open only
for n in $(gh api repos/$R/issues/$MAP/sub_issues --jq '.[] | select(.state=="open") | .number'); do
  s=$(gh api repos/$R/issues/$n            --jq '.issue_dependencies_summary.blocked_by')
  l=$(gh api repos/$R/issues/$n/dependencies/blocked_by --jq 'length')
  a=$(gh api repos/$R/issues/$n            --jq '.assignee != null')

  [ "$s" -gt 0 ] || [ "$l" -gt 0 ] && continue   # blocked: conservative, safe to skip
  [ "$a" = "true" ] && continue                   # already claimed
  echo "FRONTIER: #$n"; break
done
```

`[ "$s" -gt 0 ] || [ "$l" -gt 0 ] && continue` relies on `&&`/`||` precedence — if you prefer clarity over brevity, write it as two separate `[ … ] && continue` lines. Either way the conjunction is what matters: never treat a ticket as claimable on one endpoint alone.

**Re-verify at claim time.** A frontier picked minutes earlier may have been re-blocked in between, so the two reads are repeated immediately before `gh issue edit --add-assignee`. This matters most on the unblock path, where the listing endpoint lags for a couple of seconds.
