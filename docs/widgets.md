# Home-screen widgets with Voltra

Prayer Lock ships two home-screen widgets, **Daily Verse** and **Prayer Streak**, on iOS (WidgetKit) and Android (Glance). They are written in React with [Voltra](https://github.com/callstackincubator/voltra), which turns a JSX tree into a JSON description that native code renders. Data flows: Zustand stores → `refreshAllWidgets()` → `voltra/client.updateWidget` → App Group → widget.

Everything below was learned from broken widgets.

## Only register widgets you feed

Voltra's config plugin generates `VoltraWidgetBundle.swift` from the `widgets` array in `app.config.js`. Any registered widget the app never pushes data to shows Voltra's "Almost ready" placeholder in the iOS widget picker **forever**. We shipped two such widgets (Next Prayer, Prayer Stats) that were registered natively but never fed, and removed them. Rule: `refreshAllWidgets()` must update every registered widget.

## The root of every variant must be a gradient

`<Voltra.LinearGradient>` renders a greedy rectangle that fills the widget. A `VStack` or `HStack` root with `backgroundColor` only paints behind its own content; stacks hug their content and the host pins the root top-leading, so you get a colored box in the corner on the system background. Wrap every variant in a shared gradient root.

## No asset-catalog images in home-screen widgets

`<Voltra.Image source={{ assetName }}>` renders blank or gray in WidgetKit with the text floating over nothing. The verse widget switched from a photo to a three-stop dark gradient and looks intentional.

## `colors` must be an array

`<Voltra.LinearGradient colors={['#2C1810', '#3A2518', '#1C1210']}>`. A pipe-joined string is silently dropped by the component's `toJSON` and you get a faint black gradient. Use `start` and `end` with the typed names (`'topLeading'`), not `startPoint` and `endPoint`, which only survive through an untyped passthrough. In typed files the tuple needs `as const`.

## Center content with spacers

Without `<Voltra.Spacer />` above and below, content sits at the top of the widget.

## Never put content in `placeholder()`

iOS renders the placeholder entry `.redacted(.placeholder)`, a gray skeleton. Real content in there turns into a gray and black mess. Voltra's default returns `rootNode: nil`, which gives a clean skeleton; leave it. The widget gallery and the long-press preview use `getSnapshot(isPreview:)`, which already serves your bundled initial state unredacted. That is where default content belongs.

## Initial states are pre-rendered at prebuild

Each widget's `initialStatePath` points to a `.tsx` that default-exports a `WidgetVariants` object built with `import { Voltra } from 'voltra/server'`. The plugin renders it to `VoltraWidgetInitialStates.swift`. To regenerate without a full `expo prebuild` (which would disturb manual native setup), call the plugin's generator directly; the package `exports` map blocks the subpath, so import it by absolute file URL. Initial-state JSON is plain; brotli compression is only used for Live Activities.

## Android is a separate tree

`updateWidget` is iOS-only. Android uses `voltra/android/client`'s `updateAndroidWidget` with `VoltraAndroid` primitives and dp-size variants (110×110, 250×110, 250×250). Keep the two trees side by side and share only data selectors.

## Gate by subscription

Widgets are a premium feature. When there is no entitlement, the same refresh path pushes a small "upgrade" variant instead of leaving the widget stale.

## A note on feature flags and native code

A JS flag hid the widgets from production for months while they were being fixed. The flag gated the sync code and the settings row, but the committed `ios/` project shipped the widget extension in every build regardless. With the flag off, widgets sat dead in the iOS picker. If you hide a widget feature, remember the extension is still there.
