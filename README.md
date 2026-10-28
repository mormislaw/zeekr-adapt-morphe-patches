# zeekr-adapt Patches for Morphe

A Morphe patch bundle that ports [zeekr-adapt](https://github.com/nobleKr/zeekr-adapt) to the
Morphe patcher. zeekr-adapt adapts phone apps to automotive head units (Zeekr DHU and similar)
whose 2560×1600 screen reports a low density (160 dpi), so stock apps render a microscopic UI and
misbehave (root/emulator checks, "offline" network, blank screen in car-aware apps, no steering
wheel media control).

The patch does what `patch_manifest.py` + `patch_apk.sh` do in zeekr-adapt, inside Morphe:

| zeekr-adapt step | Morphe patch |
|---|---|
| `patch_manifest.py`: `appComponentFactory` → `com.dhuadapter.DhuAdapterFactory` | manifest edit; the app's own factory is kept and called by the adapter |
| `patch_manifest.py --provider/--receiver/--queries-package` (`--media-bridge`) | `CoverProvider`, `RecoveryReceiver`, `<queries><package com.zeekr.mediacenter/>` |
| `build.sh` + `classesN.dex` injection | extension `extensions/dhuadapter.mpe` (adapter + Pine) merged by Morphe |
| `assets/dhu-adapter-config.json` from flags | generated from the patch options |
| `tools/extract_apk_cert.py` → `assets/orig-cert.der` (`--copy-sign`) | taken from the original APK's v3.1/v3/v2 signer |
| `lib/arm64-v8a/libpine.so` | `assets/dhu-adapter/lib/<abi>/libpine.so` (arm64-v8a, armeabi-v7a), loaded at runtime |
| zipalign + apksigner (DHUAdapter key) | done by Morphe |

All hooks are the original zeekr-adapt runtime: they target only Android framework APIs and are
installed with Pine before `Application.onCreate`.

## Add to Morphe Manager

[Add this patch source](https://morphe.software/add-source?github=mormislaw/zeekr-adapt-morphe-patches)
or add `https://github.com/mormislaw/zeekr-adapt-morphe-patches` as a patch source in Morphe Manager.

## Patches

| App | Package | Preset (default options) | zeekr-adapt flags |
|---|---|---|---|
| any app | — | Display / UI only | *(none)* |
| Waze | `com.waze` | automotive de-mask | `--automotive-fix` |
| Apple Music | `com.apple.android.music` | root, emulator, network, SIM gate bypass, MediaCenter | `--root-bypass --emulator-bypass --network-bypass --sim-ready --media-bridge` |
| Spotify | `com.spotify.music` | MediaCenter, original signature | `--media-bridge --copy-sign` |
| YouTube Music | `com.google.android.apps.youtube.music` | MediaCenter | `--media-bridge` |
| Telegram | `org.telegram.messenger` | Display / UI only | *(none)* |

Every patch exposes all options, so presets can be changed. Apply only one DHU adapter patch per
app: the app specific "DHU adapter" or "DHU adapter (any app)".

<!-- PATCHES_START -->
The patch list is generated here by the first release.
<!-- PATCHES_END -->

## Options

| Option | Default | Category / effect |
|---|---|---|
| `display` | on | 1: density, dp recompute, orientation, floating button, corners, font, WebView |
| `metricsDpi`, `configDpi` | 280 | 1: rendering / layout DPI, keep them equal |
| `forceOrientation` | on | 1: `setRequestedOrientation` → UNSPECIFIED |
| `fullscreen` | off | 1: hide system bars |
| `backButton`, `backButtonOnlySettings`, `backButtonSize`, `backButtonAlpha`, `backButtonColor` | on, off, 48, 0.5, `#4CAF50` | 1: floating button and settings panel |
| `roundedCorners`, `roundedCornersRadius` | on, 16 | 1: rounded window corners |
| `fontOverride` | empty | 1: font from the app's `assets/fonts/` |
| `userAgentOverride` | Chrome 120 mobile | 1: `WebSettings.getDefaultUserAgent` |
| `rootBypass` | off | 2 |
| `emulatorBypass` | off | 3 |
| `automotiveFix` | off | 4 |
| `networkBypass` | off | 5a |
| `simReady` | off | 5b, breaks phone number registration |
| `mediaBridge` | off | 6: Zeekr MediaCenter |
| `allowCapture` | off | 7: strip `FLAG_SECURE` |
| `debug` | off | 8: `adb logcat -s DhuAdapter` |
| `copySign` | off | 9: genuine signer certificate for the app's self-check |

The DPI and the floating button can still be tuned at runtime in the settings panel (long press
on the button); those values are saved in the app and override the embedded config.

## Notes

- Use a universal APK or let Morphe merge the splits (APKM/XAPK). The arm64 native split is
  needed for apps that load their own libraries.
- `copySign` needs the original store-signed APK, otherwise the patch logs a warning and the
  spoof does nothing.
- Pine is used from Maven Central (`top.canyie.pine:core:0.3.0`), classes and native library of the
  same release, instead of the vendored smali and `libpine.so` of zeekr-adapt.
- The `--phone-ui` flag of `patch_apk.sh` is not ported: the current zeekr-adapt runtime no
  longer reads it.
- libpine.so exists for arm64-v8a and armeabi-v7a only. On other devices (e.g. an x86_64
  emulator, also when it translates an ARM-only app) Pine is skipped and logged, the app starts
  without the framework hooks; the floating button and window chrome still work.

## Build

```
./gradlew build
```

The Morphe Gradle plugin and patcher come from GitHub Packages, so set `GITHUB_ACTOR` and a
`GITHUB_TOKEN` with the `read:packages` scope (or `gpr.user` / `gpr.key` in `~/.gradle/gradle.properties`).
The bundle is written to `patches/build/libs/`.

To patch an APK locally without Morphe Manager (split bundles are merged first, the output is
unsigned, sign it with `apksigner`):

```
./gradlew :patches:patchApk -Papk=waze.apkm -Pout=waze-dhu.apk -Ppkg=com.waze -Popts=debug=true
```

`-Ppkg` selects the app specific patch (`any` for the universal one), `-Popts` sets options as
`key=value` pairs separated by commas.

## License

GPLv3 (Morphe patches template). The runtime in `extensions/dhuadapter` is derived from
zeekr-adapt (MIT). See [NOTICE](NOTICE).
