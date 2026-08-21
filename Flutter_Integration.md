# FieldTrack Tracker Flutter — Installation Guide

Everything needed to install `trackit_flutter` in a Flutter app: from an empty project
to a working background tracker on Android and iOS.

Follow the parts in order. Each one ends with a **check** — a command or observation
that proves the step worked, so a mistake is caught where it was made rather than three
steps later as an unrelated-looking error.

- [0. Before you start](#0-before-you-start)
- [1. Add the dependency](#1-add-the-dependency)
- [2. Android setup](#2-android-setup)
- [3. iOS setup](#3-ios-setup)
- [4. Wire up the Dart side](#4-wire-up-the-dart-side)
- [5. Permissions, in the right order](#5-permissions-in-the-right-order)
- [6. Device settings that decide whether background tracking survives](#6-device-settings-that-decide-whether-background-tracking-survives)
- [7. Verify the integration](#7-verify-the-integration)
- [8. Troubleshooting](#8-troubleshooting)
- [9. Version pinning](#9-version-pinning)

`README.md` is the API manual — what each method does and why. This file is the
mechanical setup. Where the two overlap, this one is the checklist.

---

## 0. Before you start

### Toolchain

| Requirement | Version | Why |
|---|---|---|
| Flutter | **3.24.0+** (built and tested on 3.38) | The plugin's `environment.flutter` floor |
| Dart | **3.5.0+** | Comes with the above |
| JDK | **17** | The AARs are compiled for JVM 17 |
| Android Gradle Plugin | **8.11.1** — *not* 9.x | See [the AGP 9 trap](#the-agp-9-trap) |
| Kotlin Gradle Plugin | **2.4.10** | The AARs are compiled with Kotlin 2.4.10 |
| Android `compileSdk` | **37** | `fieldtrack-core`'s AAR metadata demands it |
| Android `minSdk` | **26** | Below this the manifest merge fails |
| Xcode | 16+ with the iOS 17 SDK | The XCFrameworks are Swift 6 with library evolution |
| iOS deployment target | **17.0** | `@Observable`, `CLBackgroundActivitySession` |
| CocoaPods | any recent | The default Flutter iOS install path |

None of these are the plugin's choices — every one is inherited from the native SDKs.

### Credentials and artifacts to obtain first

Almost nothing. Everything the plugin needs to build ships inside the published
package: the iOS SDK frameworks are bundled, and the Android artifacts resolve
through a repository the plugin configures by itself. You never need SDK source,
SDK repository access, or any repository token.

The one thing to obtain from us:

1. **License tokens** — release builds on **both** platforms check one. iOS tokens
   are prefixed `TRACKER-`, Android tokens `TRACKIT-`. Simulator and debuggable
   builds never check them, so you can defer this until you ship. Contact your
   FieldTrack representative with your application id / bundle id (tokens are bound
   to it).

### What this plugin is

A bridge. All capture, filtering, motion detection and track plotting happen in the
native SDKs; the Dart layer moves values across a channel. Background tracking —
including after the app is killed — is **entirely native**, so you write no background
isolate, no headless callback, no foreground-service code and no boot receiver.

---

## 1. Add the dependency

In your app's `pubspec.yaml` (the exact package name and current version are in the
pub.dev listing we announce):

```yaml
dependencies:
  trackit_flutter: ^0.4.0
```

**Check:**

```bash
flutter pub get
```

resolves without errors.

---

## 2. Android setup

Two files. Every value below is required, and each has a distinct failure if missed.

### 2.1 Repository, token and the androidx pin — nothing to do

The plugin handles all three itself. Its own `android/build.gradle` carries the
JitPack read token and injects the JitPack repository **and** the androidx.core pin
into every project of your build, so your app needs no token, no repository block and
no pin. Adding the plugin to `pubspec.yaml` is the whole resolution setup.

(For CI or token rotation, `JITPACK_AUTH_TOKEN` in the environment or `authToken` in
`~/.gradle/gradle.properties` override the embedded token; with neither set, the
embedded one is used.)

#### The AGP 9 trap (handled for you — context only)

`fieldtrack-core` depends on `androidx.core 1.19.0`, whose AAR metadata declares *"requires
Android Gradle plugin 9.1.0 or higher"*. Flutter's Gradle plugin (through 3.38) **cannot
run under AGP 9** — it throws an NPE, because AGP 9 has Kotlin support built in while
Flutter still expects `org.jetbrains.kotlin.android` to be applied separately. The two
requirements are mutually exclusive as shipped.

Forcing `androidx.core` down to 1.16.0 breaks the tie, and is safe rather than merely
convenient: the only `androidx.core` APIs the SDK touches are `NotificationCompat`,
`ServiceCompat`, `ContextCompat` and the three `Location*Compat` classes, all stable
since core 1.7. The plugin injects this pin for you; it disappears when either Flutter
supports AGP 9 or the SDK lowers its floor.

### 2.2 `android/settings.gradle.kts` — plugin versions

```kotlin
plugins {
    id("dev.flutter.flutter-plugin-loader") version "1.0.0"
    id("com.android.application") version "8.11.1" apply false   // NOT 9.x
    id("org.jetbrains.kotlin.android") version "2.4.10" apply false
}
```

### 2.3 `android/app/build.gradle.kts` — SDK levels and JVM target

State `compileSdk` and `minSdk` explicitly. Do not use `flutter.compileSdkVersion` /
`flutter.minSdkVersion`: those track the Flutter defaults, which are not what the AARs
require.

```kotlin
android {
    compileSdk = 37                       // fieldtrack-core's AAR metadata demands 37
    defaultConfig {
        minSdk = 26                       // manifest merge fails below this
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

kotlin {
    // `kotlinOptions { }` is a hard error from Kotlin 2.4 — use compilerOptions.
    compilerOptions {
        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17)
    }
}

```

No extra dependencies: road snapping's HTTP client (OkHttp) ships transitively with
the SDK since 1.0.1.

### 2.4 What you do *not* add on Android

- **No manifest entries.** Every permission, the foreground service (with
  `stopWithTask="false"` and `foregroundServiceType="location"`) and all three receivers
  are declared in the AAR and merge into your APK automatically.
- **No ProGuard/R8 rules** — `consumer-rules.pro` ships in the AAR.
- **No DI framework, no Gradle plugin, no annotation processor.**

`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` is deliberately *not* declared by the SDK: it is
Play-policy sensitive and must be your own explicit decision.

**Check:**

```bash
flutter build apk --debug
```

builds. If it fails, go straight to [Troubleshooting](#8-troubleshooting) — the Android
errors are distinctive and each maps to exactly one missed step.

---

## 3. iOS setup

### 3.1 The SDK frameworks — nothing to fetch

The `TrackerCore`, `TrackerGeo` and `TrackerSnap` binary XCFrameworks ship **inside
the plugin package**; both the CocoaPods podspec and the SPM manifest vendor them from
the same directory, so either install path just works. There is no download step, no
SDK checkout, and no build script to run.

**Check:** after `flutter pub get`,
`ls ~/.pub-cache/hosted/pub.dev/trackit_flutter-*/ios/trackit_flutter/Frameworks/`
lists three `.xcframework` bundles.

### 3.2 `ios/Podfile` — the 17.0 floor

```ruby
platform :ios, '17.0'

# …existing Flutter boilerplate…

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '17.0'
    end
  end
end
```

### 3.3 The Runner target's deployment target

Set `IPHONEOS_DEPLOYMENT_TARGET = 17.0` in `Runner.xcodeproj` (Xcode → Runner target →
General → Minimum Deployments, or edit the three occurrences in `project.pbxproj`).

Flutter's template defaults to 13.0, and the build then fails with *"Compiling for
iOS 13.0, but module 'trackit_flutter' has a minimum deployment target of iOS 17.0"*.

### 3.4 `ios/Runner/AppDelegate.swift` — the one required line

```swift
import Flutter
import UIKit
import trackit_flutter

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    TrackerLaunch.ready()                          // ← BEFORE the registrant
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

**Why this exists, and why it cannot be hidden inside the plugin.** `ready()` registers
the SDK's `BGTaskScheduler` handlers, and Apple requires every launch handler to be
registered *before* `didFinishLaunchingWithOptions` returns — Dart starts after that
returns. Worse, iOS relaunches a terminated app **into the background** on a
significant-location change or region exit: no Flutter view exists then and Dart may
never run, yet `ready()` must still happen for capture to resume. Plugin registration
timing is host-configurable, so hiding this inside it would turn a silent miss into
"loses data in the field" months later.

Dart's `Tracker.instance.ready(config)` still runs later and is safe — the SDK guards
re-registration. Consequence to know: **fields affecting background-task registration
are fixed at launch on iOS**; a config passed from Dart changes capture parameters, not
registration.

### 3.5 `ios/Runner/Info.plist`

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>Records your route while you use the app.</string>

<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>Records your route in the background so trips are complete.</string>

<key>NSMotionUsageDescription</key>
<string>Motion data detects when you start and stop moving, saving battery.</string>

<key>UIBackgroundModes</key>
<array>
  <string>location</string>
  <string>processing</string>
</array>

<!-- VERBATIM. A mismatch is an ObjC exception at registration, not a warning. -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
  <string>com.fieldtrack360.tracker.backstop</string>
  <string>com.fieldtrack360.tracker.sync</string>
</array>
```

For **release** builds add the licence token (or pass it as
`TrackerConfig(license: …)`):

```xml
<key>TrackerLicense</key>
<string>TRACKER-…</string>
```

On Android the equivalent native entry is manifest meta-data named `TrackItLicense` —
**that exact legacy spelling**; the Android SDK kept the key name through the rebrand.
`TrackerConfig(license:)` sidesteps the naming question on both platforms and wins
over the native entry:

```xml
<!-- android/app/src/main/AndroidManifest.xml, inside <application> -->
<meta-data android:name="TrackItLicense" android:value="TRACKIT-…" />
```

Using `ios.requestTemporaryFullAccuracy()`? Add
`NSLocationTemporaryUsageDescriptionDictionary` with your purpose key too.

#### To debug on a physical iPhone, add two more keys

Not a Tracker requirement — a Flutter one, and Flutter's own iOS template does **not**
include them:

```xml
<key>NSLocalNetworkUsageDescription</key>
<string>Allow Flutter tools on your computer to connect and debug your application.</string>
<key>NSBonjourServices</key>
<array>
  <string>_dartobservatory._tcp</string>
</array>
```

`flutter run` locates the Dart VM Service by mDNS on iOS 14+. Without these keys the app
cannot advertise it, so the tool never connects and, after about a minute, kills the app
— surfacing as **"The Dart VM Service was not discovered after 60 seconds"** and
`App terminated due to signal 9`. The app itself is fine: a release build of the same
code installs and runs normally, which is the quickest way to confirm this is the cause.

On first debug launch iOS asks *"…would like to find and connect to devices on your
local network"*. **Allow it.** If it was denied, re-enable under
**Settings → Privacy & Security → Local Network**.

Debug-only concern: these keys make iOS show that prompt, which a shipped app should not
ask for without reason. Keep them out of release builds — a separate debug `Info.plist`,
or a run-script phase that strips them for Release.

### 3.6 Optional: `tracker.config.json`

Bundle this in the app to configure the SDK at launch, before Dart exists. It uses the
same schema as the Dart `TrackerConfig` wire format. Only needed if you must change
settings that are fixed at launch; otherwise configure from Dart.

**Check:**

```bash
cd ios && pod install && cd ..
flutter build ios --debug --no-codesign
```

builds, and the resulting `Runner.app/Frameworks/` contains `TrackerCore.framework`.

---

## 4. Wire up the Dart side

The plugin shows no UI, prompts nothing and draws nothing. These are the pieces your app
must write. A working reference for each is in the plugin's `sample_app/` sample app.

### 4.1 Startup

```dart
import 'package:trackit_flutter/trackit_flutter.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();

  // 1. Subscribe FIRST. The stream has no replay, and ready() is what reports a
  //    session that a crash or a kill left open.
  Tracker.instance.events.listen(_handleEvent);

  // 2. Start readiness, but do NOT await it here.
  final ready = Tracker.instance
      .ready(const TrackerConfig(
        service: ServiceConfig(
          notificationTitle: 'Tracking active',
          notificationText: 'Recording your route',
        ),
      ))
      .timeout(const Duration(seconds: 15));

  // 3. Paint immediately.
  runApp(MyApp(ready: ready));
}
```

**Do not `await ready()` before `runApp`.** Awaiting a platform-channel call before the
first frame means any stall on the native side appears as an app frozen on its splash
screen with nothing to explain it. Await it inside the UI, keep SDK-dependent controls
disabled until it completes (`start()` before `ready()` is a typed `NOT_READY` anyway),
and let the timeout turn a hang into a message.

### 4.2 Recover an interrupted session

A session can still be open at launch after a crash, a kill, or a reboot. Handle it both
ways, because the event ordering is easy to get wrong:

```dart
final open = await Tracker.instance.currentSession();   // pull
// …and SessionInterruptedEvent, emitted during ready() to the stream you subscribed to
```

### 4.3 Track

```dart
final session = await Tracker.instance.start(tag: 'commute');
// …later…
await Tracker.instance.stop();

final track = await Tracker.instance.buildTrack(
  query: PointQuery(sessionId: session.id),
);
```

### 4.4 Draw

The plugin returns geometry; you choose the map widget (`flutter_map`,
`google_maps_flutter`, anything). Three rules keep your render identical to the
exported track:

```dart
// 1. Decode at the payload's precision — NEVER the default 5. The SDK encodes at 6,
//    and decoding at 5 silently returns coordinates ten times too small.
final line = decodePolyline(track.encodedPolyline, precision: track.precision);

// 2. Use the arrows the SDK placed; do not compute your own.
for (final a in track.arrows) drawArrow(a.position, rotation: a.bearing);

// 3. Speed bands are green/yellow/red on BOTH platforms, traffic-map sense:
//    green is the FASTEST band, red the slowest.
Color bandColor(String? band) => switch (band) {
      'green' => Colors.green,
      'yellow' => Colors.orange,
      'red' => Colors.red,
      _ => Colors.indigo,
    };
```

### 4.5 Live tracking

```dart
Tracker.instance.liveTrack.listen((frame) {
  if (frame.sequence <= _lastDrawn) return;   // REQUIRED: frames can arrive out of order
  _lastDrawn = frame.sequence;

  final tail = decodePolyline(frame.frozenTailPolyline, precision: frame.precision);
  final puck = frame.puck;    // headingDeg is null when too slow to have one —
                              // hold your last rotation, never snap to 0°
});
```

Never re-smooth `frozenTailPolyline`; it is already the same smoothing the full track
uses. **Positions stream only inside a session** — before `start()` there is nothing to
follow, and `getCurrentLocation()` is the single-fix alternative.

### 4.6 License and security configuration

Release builds require a license token on **both** platforms — development builds
never check one, so this section only matters when you ship. The full behaviour
(offline gate, online revocation check, which statuses stop tracking) is in the
README's Licensing section; the wiring is:

```dart
await Tracker.instance.ready(const TrackerConfig(
  // Or use the native entries: TrackerLicense (iOS Info.plist) /
  // TrackItLicense (Android manifest meta-data — legacy spelling, deliberate).
  license: String.fromEnvironment('TRACKER_LICENSE'),

  // Android-only: the device-integrity policy block. Omit any field to keep the
  // SDK default. The whole layer is waived in debuggable builds.
  security: SecurityConfig(
    // hooking + mockLocation default to BLOCK, the rest to WARN.
    accessibilityAllowlist: ['com.yourco.kiosk'],
  ),

  // Android-only: base URL the optional sync module resolves upload paths against.
  baseUrl: 'https://api.example.com',
));
```

Handle the outcomes: `ready()` throws `LICENSE_MISSING` / `LICENSE_INVALID` /
`LICENSE_BUNDLE_MISMATCH` (offline gate) and `DEVICE_INTEGRITY_BLOCKED` (a
BLOCK-policy signal fired; `Tracker.instance.getIntegrity()` names the signals).
Listen for `LicenseCheckedEvent` (Android) / `LicenseDeactivatedEvent` (iOS) for the
online check, and `BatteryChangeEvent` / `IntegrityChangeEvent` alongside the rest of
the stream.

### 4.7 Errors

Every call throws `TrackerException` with a typed `code`. **Branch on the code, never
the message.**

```dart
try {
  await Tracker.instance.start();
} on TrackerException catch (e) {
  switch (e.code) {
    case TrackerErrorCode.permissionDenied:      // walk the ladder
    case TrackerErrorCode.locationDisabled:      // send them to Settings
    case TrackerErrorCode.notImplementedOnPlatform:  // other platform has it, this one not yet
    default:
  }
}
```

---

## 5. Permissions, in the right order

The SDK decides *which* permissions the running OS version needs and in what order; your
app decides *when* to ask and supplies the rationale. Nothing is requested
automatically. Prompting happens natively — the ladder cannot be walked correctly from
Dart.

**Ask for foreground location when you first need a position** (for example to centre a
map), and leave the rest to a screen where the user has context. Requesting everything
at launch is the reliable way to be denied permanently — Android silently denies
background location if it is requested alongside fine location.

```dart
// After the first frame: prompting needs an attached Activity on Android.
final snapshot = await Tracker.instance.getPermissions();
if (snapshot.tier == PermissionTier.none) {
  await Tracker.instance.requestForegroundPermissions();
}
```

The full ladder:

| Step | Android | iOS |
|---|---|---|
| 1 | `requestNotificationPermission()` (API 33+, ask before starting the service) | not applicable — resolves as success |
| 2 | `requestForegroundPermissions()` — fine + coarse together | When-In-Use |
| 3 | `requestBackgroundPermission()` — **prompt only on Android 10**; on 11+ read `background.kind == needsSettings` and call `openAppSettings()` | Always, only reachable from When-In-Use |
| 4 | `requestActivityRecognitionPermission()` — optional | `ios.requestMotionPermission()` — optional |

Read `PermissionSnapshot.background.kind` before asking for background; it tells you
whether a prompt exists at all. Platform-inapplicable fields are **null, not false** — a
null `hasNotification` on iOS means "not a concept here", not "denied".

Background location is required for geofencing on Android; without it the SDK refuses to
arm fences with `GEOFENCE_REGISTRATION_FAILED`.

---

## 6. Device settings that decide whether background tracking survives

Getting the code right is not sufficient on every device.

### Android OEM power management

"Capture survives a swipe-kill" is what **stock** Android does, and what the service is
configured for. Xiaomi/Redmi (MIUI/HyperOS), Oppo/Realme, Vivo, Huawei and to a lesser
degree Samsung override this and kill the process — foreground service included — when
the task is swiped away. No SDK can defeat this from inside the app.

Ask your users to set, per app:

| Setting | Where (names vary by skin) |
|---|---|
| **Autostart / Auto-launch** — enable | Settings → Apps → *your app* → Autostart |
| **Battery** → *No restrictions* / *Don't optimise* | Settings → Apps → *your app* → Battery |
| **Lock in Recents** (padlock on the task card) | Recents screen |

Symptoms when unset: the notification vanishes seconds or minutes after the app is
swiped away, the session stays open with a gap in its points, and the next launch
reports it via `SessionInterruptedEvent`. MIUI also refuses background service starts
for apps without Autostart, which looks like "the service never started".

For contrast, measured on a clean Android 16 emulator: the foreground service is up
~0.4 s after `start()` and survives a swipe-kill issued 150 ms later.

### iOS

After a user **force-quits** (swipe up in the app switcher), iOS does not reliably
relaunch the app for background activity. This is OS policy, not an SDK gap — surface it
in your tracking-health UI and never build correctness on it. Significant-location
relaunch is coarse (~500 m, minutes) and is a recovery trigger, not a capture path.

---

## 7. Verify the integration

Work through these on a real device. Each one exercises a different part of the stack.

| # | Test | Expected |
|---|---|---|
| 1 | Launch the app | No splash hang; UI appears immediately |
| 2 | Grant foreground location, tap start | Android: tracking notification appears within ~1 s |
| 3 | Walk/drive a few minutes with the app open | Live puck follows; points accumulate |
| 4 | Background the app for 10+ minutes | Capture continues; full trace on return |
| 5 | Swipe the app from Recents mid-session | Android: notification stays, capture continues. iOS: capture pauses, then relaunches on a significant-location change |
| 6 | Reopen the app | `currentSession()` finds the open session; `buildTrack()` draws the trace including the killed period |
| 7 | Stop, then draw the finished track | Polyline, stops, arrows and stats render; nothing lands in the ocean (that means precision 5) |
| 8 | Reboot the device mid-session | Android: capture resumes per `ServiceConfig.startOnBoot` |
| 9 | Revoke location in Settings while tracking | A `ProviderChange` event arrives; the app degrades rather than crashing |

---

## 8. Troubleshooting

### Android build

| Symptom | Cause | Fix |
|---|---|---|
| `Could not find com.github.fieldtrack360.fieldtrack:fieldtrack` | A stale `flutter clean`/pub state, or an `authToken`/`JITPACK_AUTH_TOKEN` override set to a wrong value (they beat the plugin's embedded token) | Unset the override, `flutter clean && flutter pub get` |
| `checkDebugAarMetadata`: *requires minCompileSdk 37* | `compileSdk` below 37 | [2.3](#23-androidappbuildgradlekts--sdk-levels-and-jvm-target) |
| Manifest merge fails naming `fieldtrack-core` | `minSdk` below 26 | [2.3](#23-androidappbuildgradlekts--sdk-levels-and-jvm-target) |
| *class file was compiled with a newer version of Kotlin* | Kotlin plugin below 2.4.10 | [2.2](#22-androidsettingsgradlekts--plugin-versions) |
| *androidx.core:core:1.19.0 requires Android Gradle plugin 9.1.0 or higher* | The plugin's injected androidx pin was overridden by a host resolutionStrategy | Remove the conflicting force; the plugin pins core 1.16.0 itself |
| NullPointerException inside Flutter's Gradle plugin | AGP 9.x | Use AGP 8.11.1 |
| `kotlinOptions` unresolved | Kotlin 2.4 removed it | Use `compilerOptions { }` |

### iOS build

| Symptom | Cause | Fix |
|---|---|---|
| *Compiling for iOS 13.0, but module 'trackit_flutter' has a minimum deployment target of iOS 17.0* | Runner target still at the Flutter default | [3.3](#33-the-runner-targets-deployment-target) |
| Undefined symbols `TrackerCore` at link time | A corrupted pub cache dropped the bundled frameworks | `flutter clean && dart pub cache repair && flutter pub get`, then [3.1](#31-the-sdk-frameworks--nothing-to-fetch)'s check |
| ObjC exception from `BGTaskScheduler.register` at launch | `BGTaskSchedulerPermittedIdentifiers` missing or misspelled | [3.5](#35-iosrunnerinfoplist) |
| Killed-app tracking never resumes | `TrackerLaunch.ready()` missing or after the registrant | [3.4](#34-iosrunnerappdelegateswift--the-one-required-line) |
| `LICENSE_MISSING` in a TestFlight/App Store build | No licence token | [3.5](#35-iosrunnerinfoplist) |
| **Unable to Install** … *Failed to verify code signature of …/Frameworks/`<some>`.framework* `0xe800801c (No code signature found.)` | A framework from a **package you removed** is still in the old build output and gets copied into the bundle. iOS verifies every framework in the bundle, so one orphan fails the whole install | Clean, purge Pods, reinstall — see [below](#after-removing-a-dart-package-clean-the-ios-build) |
| *The Dart VM Service was not discovered after 60 seconds* | Usually **not** a networking or Automation-permission problem: the install failed or the app crashed, so no process ever published a VM service | Read the real error in Xcode's log, or check `xcrun devicectl device info apps --device <id>` — if your bundle id is absent it was never installed |
| `pod install` dies with *Unicode Normalization not appropriate for ASCII-8BIT (Encoding::CompatibilityError)* | Ruby cannot normalise the project path without a UTF-8 locale. Likely when the path has spaces or non-ASCII characters | `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 pod install`, or `export LANG=en_US.UTF-8` in your shell profile |

#### After removing a Dart package, clean the iOS build

Deleting a dependency from `pubspec.yaml` does not remove its already-built framework
from `build/` or `ios/Pods/`. Incremental builds keep embedding it, unsigned, and the
device rejects the bundle — with an error naming a package you no longer depend on,
which is why this one is so confusing.

```bash
flutter clean
rm -rf ios/Pods ios/.symlinks ios/Flutter/ephemeral
flutter pub get
cd ios && pod install && cd ..
```

Check `ios/Podfile.lock` no longer mentions the package, and that
`build/ios/iphoneos/Runner.app/Frameworks/` contains only frameworks you still use.

### Runtime

| Symptom | Cause | Fix |
|---|---|---|
| App frozen on the splash screen | `await ready()` before `runApp` | [4.1](#41-startup) |
| `MissingPluginException` | Hot reload after adding the plugin | Full restart — hot reload does not register new plugins |
| The track is in the ocean, ~10× off | Polyline decoded at precision 5 | Pass `track.precision` / `frame.precision` |
| Segments all one colour | Matching `slow`/`medium`/`fast` | The values are `green`/`yellow`/`red` |
| `NOT_IMPLEMENTED_ON_PLATFORM` from `getIntegrity`/`checkIntegrity`/`getLicenseInfo` on iOS | The iOS SDK has not shipped these layers | Expected — Android-only today |
| `FIX_TIMEOUT` from `getCurrentLocation` on iOS | The iOS SDK collapses timeout, missing authorization and concurrent calls onto this one code | Read `message`; do not treat it as "retry later" |
| Background permission never grantable; snapshot keeps saying `prompt` | Android 11+ shows no runtime prompt | Read `background.kind`; call `openAppSettings()` on `needsSettings` |
| `GEOFENCE_REGISTRATION_FAILED` | Background location not granted (Android) | Complete the ladder first |
| Notification disappears after a swipe-kill | OEM power management | [6](#6-device-settings-that-decide-whether-background-tracking-survives) |
| Emulator stores only 1–3 points from a dozen injected fixes | Injected fixes carry no hardware speed or bearing, so the pipeline treats them as network-derived and skips most | Working as intended — validate on a real drive |

### Development workflow

**After changing anything native, rebuild — do not rely on `flutter install`.** It can
reinstall a stale APK, leaving a Dart half and a native half from different builds, which
produces symptoms that match no source you are reading:

```bash
flutter build apk --debug && adb install -r build/app/outputs/flutter-apk/app-debug.apk
# or simply: flutter run
```

---

## 9. Version pinning

Pin the plugin version in your `pubspec.yaml` — that is the only version you manage.
The native SDK versions are baked into each plugin release (the Android maven
coordinate inside the plugin, the iOS frameworks bundled with it), so upgrading the
SDKs is always just a plugin version bump on your side:

```yaml
dependencies:
  trackit_flutter: 0.4.0   # exact pin for reproducible builds
```

Read the CHANGELOG before bumping — SDK behaviour changes ride plugin releases.
