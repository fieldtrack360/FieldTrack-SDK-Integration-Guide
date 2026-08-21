# FieldTrack SDK — Integration Guides

Background location tracking and track plotting for mobile apps. This repository holds the
public integration guide for each supported platform: install, configuration, the full API
surface, permissions, and troubleshooting.

## Guides

| Platform | Guide | Package |
|---|---|---|
| Android | [`Android_Integration.md`](Android_Integration.md) | `com.github.fieldtrack360.fieldtrack` (JitPack) |
| iOS | [`iOS_Integration.md`](iOS_Integration.md) | `https://github.com/fieldtrack360/tracker-ios` (SwiftPM) |
| React Native | [`React-Native_Integration.md`](React-Native_Integration.md) | [`@fieldtrack360/react-native-tracker`](https://www.npmjs.com/package/@fieldtrack360/react-native-tracker) |
| Flutter | [`Flutter_Integration.md`](Flutter_Integration.md) | `trackit_flutter` (pub.dev) |

## Platform requirements

| | Android | iOS |
|---|---|---|
| Floor | `minSdk` 26 (Android 8.0) | iOS 17.0 |
| Toolchain | AGP 8.x · Kotlin 2.0+ · JDK 11+ | Xcode 26+ · Swift 6 · CocoaPods 1.15+ |
| Compile / target | `compileSdk` / `targetSdk` 36 | — |

React Native additionally requires RN **0.81+** with the New Architecture enabled and Node 22+.
None of these floors is adjustable.

## What the SDK does

Records a location trace that survives backgrounding and process termination, filters it
through a Kalman filter and a staged acceptance pipeline, and turns the result into a plotted
track you can render.

Optional modules, available on every platform:

- **Maps** — render a finished track and a live one
- **Snap** — OSRM road matching
- **Sync** — upload to your backend with a retry queue
- **Geofences** — enter/exit callbacks

## Licensing

Tracker is licensed software. Buy a licence or start a free trial at
<https://fieldtrack360-sdk.devstree.in/>.

- **One key covers both platforms** — you supply a single `license` token.
- **A key is bound to one application identifier**, named at checkout. Decide your
  `applicationId` / bundle identifier before you buy; a token issued against a different
  identifier is rejected.
- The token is supplied one way: the `license` field on the config passed to `ready()`.
- On Android, debuggable builds are waived automatically — you can develop with no token.

Keep the token out of source control (`local.properties`, `.env`, or your CI secret store).

## Where to start

1. Open the guide for your platform.
2. Follow **Install** and the host configuration section — background modes, manifest entries,
   and permission strings must be in place before the first run.
3. Call `ready()` with your config, then request permissions in the order the guide gives.
4. Use the **Troubleshooting** section at the end of each guide when something does not start.
