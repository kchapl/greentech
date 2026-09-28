# Scala 3 / Scala.js stack survey for greentech

**Research ticket:** [Survey the Scala 3 and Scala.js stack options](https://github.com/kchapl/greentech/issues/11)  
**Map:** [greentech: spec for a mirror-and-forecast green-tech investing app](https://github.com/kchapl/greentech/issues/9)  
**Accessed:** 2026-09-28  
**Scope:** Scala 3 + Scala.js, single-user market dashboard, daily OHLCV/candlestick charts, UK/European listings primary and US listings secondary.

## Finding

Scala.js has an active Scala 3 path with official sbt documentation, and Vite has an official Scala.js plugin. UI choices worth carrying to the human-facing front-end decision are:

1. **Laminar** — Scala.js-native, Functional Reactive Programming UI library. Low ceremony, typed DOM, direct DOM bindings, and official Scala.js tutorials demonstrate it with Scala 3, sbt, Vite, and Chart.js. Strong default for an all-Scala frontend.
2. **Tyrian** — Elm-inspired UI library for Scala 3. Its explicit Model/Msg/update/view and subscriptions/commands shape is useful for a stateful dashboard. It is maintained in the Indigo Engine monorepo. The current Tyrian site lists `0.30.0-M6`, a milestone/pre-release-looking version; verify release maturity/maintenance and dependencies before choosing.
3. **scalajs-react** — Scala.js bindings to React, with access to React's wider JS component ecosystem and optional integrations. The upstream currently requires React >=19, Scala >=2.13, and Scala.js >=1.20; the releases page lists v4.0.0 as latest. Strongest if React ecosystem/interoperability dominates, but brings React's API and JS package stack into the app.

**Build recommendation to take into the decision:** use **sbt + Scala.js + Vite** as the baseline to prototype. This is the most completely documented route in the official Scala.js project tutorials, and it has a Scala 3 project template. It gives incremental Scala.js linking, browser HMR/live reload through Vite, and a normal npm production build. Mill is a viable alternative with first-party Scala.js modules and Scala 3 web examples; the project would need to assemble its own UI / JS bundling conventions. Scala CLI supports Scala.js compile/run/test/package but its own guide says it does not bundle JS libraries, so is better suited to small projects or experiments unless paired with separate bundling infrastructure.

**Chart recommendation:** keep chart integration behind a small Scala.js facade/component boundary and make the later front-end decision compare:

- **TradingView Lightweight Charts** for a focused, high-performance financial/candlestick chart. It provides a native candle series and update APIs. Its Apache 2.0 licensing has an extra explicit product attribution requirement: include the notice and a TradingView link on the public page. No first-party Scala facade was found in this survey, so likely use ScalablyTyped if generation supports the npm package cleanly, or write/maintain a minimal handwritten facade.
- **Apache ECharts** for a broader dashboard charting toolkit with a built-in candlestick series, data-oriented chart composition, and Apache 2.0 license. It is a general-purpose visualization library, likely larger/broader than needed for a focused price chart. It also needs Scala.js interop via ScalablyTyped or a custom facade; the Scala.js official facade catalog did not list a dedicated ECharts facade.

The official Scala.js + Laminar tutorial shows Chart.js interop through ScalablyTyped, which reads TypeScript definitions and generates Scala.js facades. This validates the general interop route, **not** that the current versions of Lightweight Charts or ECharts will generate a convenient/sound facade without fixes. Confirm the selected chart library's TypeScript definitions, generated facade compatibility, lifecycle cleanup, resize behavior, OHLC ordering, and licensing in a short spike before settling it.

This ticket surveys the options; it does **not** lock the app's final framework or chart library. The downstream human-facing decision ticket is [Choose the front-end stack](https://github.com/kchapl/greentech/issues/16).

## Comparison

| Area | Option | Upside | Trade-offs / risks |
|---|---|---|---|
| Build + bundling | **sbt + Scala.js + Vite** | Official tutorial covers Scala 3, `fastLinkJS`, ES modules, module splitting, Vite development reload, and production bundling. Scala.js ecosystem's most documented path. | Two toolchains (JVM/sbt and Node/Vite/npm); set up npm and sbt tasks coherently in CI. |
| Build + bundling | **Mill + Scala.js module** | First-party Mill docs cover Scala.js modules, tests, `fastLinkJS`/`fullLinkJS`, JS/JVM web integration and Scala 3. Suitable if the team prefers Mill. | Fewer end-to-end Scala.js UI/bundler tutorials/templates than sbt + Vite; choose and wire npm bundling and dev reload deliberately. |
| Build + bundling | **Scala CLI** | Low setup for compiling, running, testing and packaging Scala.js; convenient small-project workflow. | Official Scala CLI guide explicitly says it has no JS bundler support; npm bundling must be assembled separately for libraries such as charts. |
| UI | **Laminar** | Scala 3-compatible Scala.js UI; typed DOM and FRP signals; no React/VDOM dependency. Official Scala.js tutorial demonstrates Laminar + Vite + Chart.js and component lifecycle hooks. | Smaller JS component ecosystem than React; custom interop likely for UI widgets/charts. FRP model has a learning curve. |
| UI | **Tyrian** | Scala 3 Elm-inspired architecture; model/update/messages and subscriptions make dashboard events/state explicit. | Current site advertises `0.30.0-M6`; check pre-release maturity, current docs/dependency compatibility, and interop examples before adoption. |
| UI | **scalajs-react** | Scala wrapper over React, including hooks, routing, testing utilities, and broad React component availability; current v4 release and React 19 support. | React semantics and more JS dependencies; adds framework surface compared with a Scala.js-native UI. |
| Financial charts | **TradingView Lightweight Charts** | Purpose-built and performant financial chart library; candlestick, bar, area, baseline, histogram, line series; supports setting/updating series data. | Attribution is mandatory on a public site: include the NOTICE attribution and a TradingView link. No dedicated Scala facade identified; generated/custom facade needs a spike. |
| Charts | **Apache ECharts** | General visualization suite; documented candlestick series, dataset model, broad chart types; Apache 2.0. | Broader/heavier library than a focused financial chart. No dedicated Scala facade identified; generated/custom facade needs a spike. |
| JS interop | **ScalablyTyped** | Converts TypeScript declaration files into Scala.js facade types; shown in official Scala.js tutorial with Chart.js. | Build plugin + npm/TypeScript toolchain and generated types add build complexity; generation quality is per-package and unverified for the chart candidates. |
| JS interop | **Handwritten facade** | Narrow, intentional API; avoids pulling a large generated surface into Scala. | Must be maintained as the npm library API evolves; isolate dynamic/JS types at the boundary and test behavior. |

## Suggested decision process

The next ticket, **Choose the front-end stack**, can decide a first candidate rather than prematurely committing in this research ticket. Suggested baseline: **sbt + Scala.js + Vite + Laminar**. For chart choice, compare Lightweight Charts vs ECharts by making the same small daily candlestick prototype: UK/EU date handling, zoom/range selection, volume, hover OHLC values, resize/lifecycle cleanup, and market up/down colors. Include the Lightweight Charts attribution requirement in the decision. Keep chart API types behind a small facade to avoid coupling the domain model to a JS library.

Questions to verify in the spike:

- Does ScalablyTyped currently generate usable Scala 3 facades from the selected chart library's TypeScript declarations, or is a handwritten facade more maintainable?
- Does the chart preserve the required OHLCV field ordering, listing-local session date, and currency display without implicit conversion?
- Is the library's license and attribution workable for the eventual deployment?
- Is interaction smooth at the historical bar counts required by the separate backtesting-validity research?

## Primary sources

1. Scala.js, [Documentation](https://www.scala-js.org/doc/index.html) — Scala.js is managed by the build tool; officially lists sbt, Scala CLI, Mill, and Gradle plugin as supported build tools.
2. Scala.js, [Basic tutorial](https://www.scala-js.org/doc/tutorial/basic/index.html) — current sbt plugin version in tutorial, Scala 3 example configuration, `fastLinkJS`, DOM facade.
3. Scala.js, [Scala.js + Vite tutorial](https://www.scala-js.org/doc/tutorial/scalajs-vite.html) — Scala 3 frontend baseline, Vite setup, `@scala-js/vite-plugin-scalajs`, hot reload and production build.
4. Scala.js, [Build UIs with Laminar](https://www.scala-js.org/doc/tutorial/laminar.html) — Laminar FRP approach, typed DOM and UI state.
5. Laminar, [Documentation](https://laminar.dev/documentation/) and [GitHub](https://github.com/raquo/Laminar) — current docs, Scala.js 1.16+ requirement, MIT license, UI model/features.
6. Tyrian, [official documentation](https://tyrian.indigoengine.io/) and [subscriptions guide](https://tyrian.indigoengine.io/documentation/guides/subs.html) — Scala 3 Elm-inspired UI, subscriptions emitting messages; homepage lists current `0.30.0-M6`.
7. Purple Kingdom Games, [Indigo Engine monorepo](https://github.com/PurpleKingdomGames/indigoengine) — current Tyrian source home and Scala 3 designation.
8. Scala.js, [Integrate JavaScript libraries with ScalablyTyped](https://www.scala-js.org/doc/tutorial/scalablytyped.html) — Chart.js example; converter reads TypeScript definitions and generates typed Scala.js APIs.
9. Scala.js, [JavaScript library facades](https://www.scala-js.org/libraries/facades.html) — existing facades and note that ScalablyTyped provides additional generated facades.
10. Mill, [Scala Web Project Examples](https://mill-build.org/mill/scalalib/web-examples.html) — Scala 3 + `ScalaJSModule`, test, link, and server integration examples.
11. Scala CLI, [Scala.js guide](https://scala-cli.virtuslab.org/docs/guides/advanced/scala-js/) — compile/run/test/package support and explicit no-JS-bundler caveat.
12. scalajs-react, [GitHub](https://github.com/japgolly/scalajs-react) and [releases](https://github.com/japgolly/scalajs-react/releases) — React wrapper, Scala/Scala.js/React requirements, latest v4 release.
13. TradingView, [Lightweight Charts docs](https://tradingview.github.io/lightweight-charts/docs) — client-side requirement, candlestick series, set/update data operations, licensing/attribution notice.
14. TradingView, [Lightweight Charts NOTICE](https://github.com/tradingview/lightweight-charts/blob/master/NOTICE) and [LICENSE](https://github.com/tradingview/lightweight-charts/blob/master/LICENSE) — TradingView copyright/link notice and Apache License 2.0.
15. Apache ECharts, [Handbook](https://echarts.apache.org/handbook/en/get-started/) and [candlestick series documentation](https://github.com/apache/echarts-doc/blob/master/en/option/series/candlestick.md) — browser installation and candlestick series semantics; [license](https://github.com/apache/echarts/blob/master/LICENSE).
