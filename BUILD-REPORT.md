# iNet — Ray-Ban Bridge v1 Skeleton: Build Report

**Date:** 2026-10-17 · **App:** iNet (package `com.skynet.raybanbridge`, versionCode 2, versionName `0.2-v1-mock`)
**Spec:** `../PRODUCT.md` (Kenn's product definition — source of truth) and `../SPEC.md`.

## ✅ Installable debug APK (built tonight)

```
/home/hatch/workspace/rayban-bridge/app/manual-build/app-debug.apk
```

- 25,530,255 bytes, signed with a generated Android debug key (`apksigner verify` passes).
- Badging verified: `application-label: 'iNet'`, `targetSdkVersion: '35'`, package `com.skynet.raybanbridge`.
- Contents verified: `classes.dex` (7.2 MB), **24 native `.so` libraries under `lib/arm64-v8a/`** (DAT core + fbjni/soloader), launcher icons at all densities + adaptive icon, mock assets (`mock_feed.mp4`, `mock_still.png`).
- Compiled manifest verified to carry the DAT meta-data `com.meta.wearable.mwdat.APPLICATION_ID = 2468383600319627` (the real "Skynet Bridge" / iNet Developer Center project ID, as one clearly-marked constant in `app/src/main/AndroidManifest.xml`).

## Build method — read this before rebuilding

The Gradle project in this directory (`./build.sh assembleDebug`) is the **canonical build** and is complete and correct for a normal machine. **In this sandbox, Gradle cannot run**: the client fails its daemon handshake with "Could not receive a message from the daemon" even though the daemon starts, listens on 127.0.0.1, and raw localhost TCP between other processes works fine (verified by direct test; three attempts incl. `--no-daemon` and reduced workers — same failure; daemon logs in `manual-build/build-run*.txt` and the toolchain gradle-home).

The APK above was therefore produced by the sanctioned fallback: a manual pipeline using the toolchain's own tools (`manual-build/build-manual.sh`, rerunnable):

1. Recursive POM resolution + artifact download (`manual-build/resolve_deps.py`): 55 artifacts, mirroring the Gradle graph (DAT 1.0.1 from the local repo; the rest from Google Maven / Maven Central). Where versions conflicted, first-seen won (a real Gradle build mediates to newest — noted below).
2. `aapt2 compile/link` (build-tools 35.0.0, platform android-35), app resources + manifest + assets.
3. `kotlinc` — the real **Kotlin 2.2.21** embeddable compiler from Maven Central (`manual-build/tools/`), matching the project's pinned Kotlin version. (The Gradle distro's bundled 2.0.21 compiler was tried first and **cannot read the DAT libraries' Kotlin 2.2 metadata**.)
4. `d8` dexing of the app + all 51 dependency jars; `zipalign`; `apksigner` (debug key).

Differences vs. the Gradle build, stated honestly:
- No AGP manifest merging. Verified harmless for the DAT modules: **none of the five DAT AARs declare activities/services/providers/receivers**. A few transitive AndroidX AARs (profileinstaller, startup-runtime) declare optional init components in their manifests; AGP would merge them, the manual APK omits them — they are self-initialization helpers, not required for this app to run.
- Dependency mediation is first-seen rather than newest-wins (e.g. coroutines resolves to 1.6.4 via DAT's graph instead of the catalog's version). The Gradle build on a normal machine produces the mediated graph.
- R8 emitted Kotlin-metadata rewrite warnings (its bundled version predates Kotlin 2.2 metadata); dexing completed and the warnings are informational for a debug build.

## Architecture

All app code is original, in `app/src/main/java/com/skynet/raybanbridge/`:

| File | Role |
|---|---|
| `BridgeApplication.kt` | Calls `Wearables.initialize(context)` once at startup, logs failures. |
| `MainActivity.kt` | Single status screen. On real hardware it's status-only; in mock mode it carries clearly-labeled **MOCK STAND-IN** buttons (arm mock listener, inject long-press, inject transcript, mock photo tap, disarm). Nothing simulated is presented as real. |
| `bridge/BridgeController.kt` | The state machine: `DISARMED → IDLE_LISTENING → STREAMING`. Arms a DAT session + inputs only; on long-press ON attaches camera + speech and opens an archive session; on long-press OFF performs a hard stop (speech stop, camera stop, collectors cancelled, stream files closed, manifest written, files queued for upload) and returns to idle. |
| `bridge/ActivationGesture.kt` | The single mapping from raw input events to Kenn's toggle semantic (see gesture findings). |
| `bridge/BridgeService.kt` | Minimal foreground service (type `connectedDevice`) that exists only while STREAMING, so the session survives in Kenn's pocket. |
| `conversation/SkynetEndpoint.kt` | Conversation-turn interface (transcript + JPEG context frame) and **`StubSkynetEndpoint` — a labeled stub** returning a canned echo reply. No live Skynet endpoint is wired in v1. |
| `conversation/SpeechOutput.kt` | `AndroidTtsSpeaker` — platform TTS as a **labeled stand-in** for the locked Skynet voice (which will arrive as audio from the future server endpoint); log-only fallback. |
| `archive/ArchiveStore.kt` | Local archive mirroring the Drive structure: `<external files>/SkynetRayBanArchive/{Images,Videos,Voice,Text,Miscellaneous}/session-<timestamp>/` + `session.json` manifest per session. |
| `archive/ArchiveUploader.kt` | Designed upload seam: `ArchiveUploader` interface + `LocalQueueUploader`, which records finished files in `pending-uploads.txt` against Drive root folder `1Jl2eRC-pncciDRor8i1tSdJsMxVYvQXu`. **It uploads nothing in v1 and says so in its logs** — no fake uploads. |

Streaming details: video is requested as compressed HEVC (MEDIUM, 24 fps) and appended raw to `Videos/stream.h265` — v1 stores the elementary stream; MP4 muxing is a later step. Stream audio (PCM 16 kHz mono, explicitly configured — no audio flows without a codec set) appends to `Voice/stream-audio.raw`. Every final transcript captures a context still via `capturePhoto()` into `Images/turn-N.jpg` and queues it for upload. Transcripts append to `Text/transcript.txt`.

## Gesture findings — Kenn's long-press EXISTS in the SDK (verified against mwdat-inputs 1.0.1 binaries)

- Attach: `DeviceSession.addInputs(InputsConfiguration(sources, consumeBack))` — an extension in `com.meta.wearable.dat.inputs`, returning `DatResult<Inputs, DeviceSessionError>`; the config type is `com.meta.wearable.dat.inputs.types.InputsConfiguration`. Capabilities attach only after the session reports `DeviceSessionState.STARTED`.
- Events arrive on `Inputs.events: Flow<InputEvent>`. A long press is **`InputEvent.Capture` with `pressType == CapturePressType.HOLD`** (siblings: `SHORT_PRESS`, `DOUBLE_PRESS`). Sources include `InputSource.CAPTOUCH` (the frame touchpad — Kenn's gesture) and `InputSource.CAPTURE_BUTTON`; the app accepts HOLD from both frame-mounted sources and ignores taps (`InputEvent.Select`), swipes (`InputEvent.Nav`), drags, and button events for activation. **No fallback mapping was needed** — the exact gesture Kenn specified is a first-class SDK event.
- Mock injection for testing: `MockGlasses.services.input.capture(CapturePressType.HOLD)` (also `services.captouch.tapAndHold()`).
- Session/capability results are **not** sealed success/failure types: `com.meta.wearable.dat.core.types.DatResult<T,E>` is a result wrapper (`isSuccess`, `getOrNull()`, `errorOrNull()`, `onSuccess`, `onFailure`). `Wearables.createSession(...)`, `addCamera`, `addInputs`, `addSpeech`, `Stream.start()`, `Stream.capturePhoto()`, `Speech.start()`, and `MockDeviceKit.pairGlasses(...)` all return it.

## Camera / speech APIs used (mwdat-camera / mwdat-speech 1.0.1)

- `DeviceSession.addCamera(StreamConfiguration(audioCodec, videoQuality, frameRate, compressVideo))` (config type in `com.meta.wearable.dat.camera.types`) → `DatResult<Camera, …>`. `Camera.stream` exposes `videoStream: Flow<VideoFrame>`, `audioStream: Flow<AudioFrame>`, `errorStream`, `state: StateFlow<StreamState>` (live value is `StreamState.STREAMING`), `start(): DatResult`, and `capturePhoto(): DatResult<PhotoData, CaptureError>` (suspend). `PhotoData` is `Bitmap(android.graphics.Bitmap)` or `HEIC(ByteBuffer)` — HEIC is decoded and re-encoded to JPEG for the endpoint/archive.
- `DeviceSession.addSpeech(SpeechConfiguration())` → `DatResult<Speech, …>`; `Speech.start(): DatResult`, `Speech.stop()`, `transcriptions: StateFlow<TranscriptionResult?>` (values are nullable; final results drive conversation turns). Mock speech: `services.speech.simulateTranscription(text, isFinal, confidence)`.

## Target device decisions (Galaxy Z Fold7, SM-F966U1, Android 17 / API 37)

- **compileSdk / targetSdk stay 35.** The installed SDK platforms are 34/35, and a target-35 app installs and runs on Android 17; targeting 37 is impossible until platform 37 is installed, and nothing in DAT 1.0.1 requires it. minSdk 31 (DAT 1.x minimum). No single-display assumption anywhere (no portrait lock; the foldable is handled by the platform).
- **16 KB page size (Android 15+ requirement): PASS.** All `.so` files bundled by the DAT modules (they ship only in `mwdat-core`) have `PT_LOAD` alignment `0x4000` (16 KB) — verified by parsing the ELF program headers. The 24 arm64 libraries in the APK come from that set.
- Kenn's side of the gate is already open (per his 01:48 note): Developer Mode enabled in the Meta AI app, glasses renamed "iNet", DAT SDK 1.0.0.0 listed.

## Launcher icon / naming

- Display label is **"iNet"** (`android:label`), per Kenn's naming (Eye + Internet).
- The Kenn-approved icon art at `app/icon/` (webp) was converted with PIL into: legacy `ic_launcher.png` at mdpi→xxxhdpi (48–192 px) and an adaptive icon (`res/mipmap-anydpi-v26/ic_launcher.xml`, white background + foreground PNG at 432 px with the art inside the safe zone). Wired via `android:icon`/`android:roundIcon`. Verified present in the APK.

## What works in mock mode (code-complete; runtime proof needs a device/emulator — none exists in this sandbox)

Arm (mock glasses pair via Mock Device Kit) → session STARTED → inputs attach → inject HOLD → STREAMING: mock camera frames counted and written to the archive, speech attaches → inject a final transcript → context photo captured → stub reply logged, archived, and spoken via platform TTS → inject HOLD again → hard stop, manifest written, files queued for Drive. The full loop compiles against the real SDK and is exercised entirely through real SDK calls on the mock device — the only simulated elements are the device itself and the two labeled stand-ins (endpoint stub, platform-TTS voice).

## What waits on later steps

1. **CLIENT_TOKEN**: exists on the iNet project's Developer Center Configuration page (starts `AR|2468383600319627|…`). No DAT 1.0.1 module required it at compile/init time for this skeleton; if registration on real hardware demands it, it belongs in the manifest beside the APPLICATION_ID meta-data. Value not copied anywhere by design.
2. **Real-hardware session test** on the Fold7 + glasses (Developer Mode already on): real HOLD gesture, real stream, real speech.
3. **Live Skynet endpoint** (replaces `StubSkynetEndpoint`) + **locked Skynet voice** audio replies (replaces the TTS stand-in).
4. **Drive upload implementation** behind the `ArchiveUploader` interface (needs Google sign-in on the phone).
5. **Gradle build on a normal machine** (`./build.sh assembleDebug`) to produce the canonical AGP artifact with full manifest merging and newest-wins mediation.
6. MP4 muxing of the archived HEVC stream; on-device runtime verification of the mock loop.

## FIX 2026-10-17 — v0.2.1-v1-mock-fix (versionCode 3): app classes were missing from the shipped APK

**Symptom (on Kenn's Fold7):** v0.2 installed fine (valid manifest, launcher icon "iNet" present) but died instantly on tap — the launcher activity class did not exist in the installed app.

**Root cause — packaging bug in `manual-build/build-manual.sh`, step 5, not an app-source bug.** d8's output for this project is **multidex**: the method count across the app + 51 dependency jars exceeds a single dex, so step 4 produced two files in `out/dex/`:
- `classes.dex` (7,163,564 bytes) — library code only; `grep -a -c "skynet"` on it returns 0.
- `classes2.dex` (7,005,036 bytes) — contains the app's own classes, including `Lcom/skynet/raybanbridge/MainActivity;` and `Lcom/skynet/raybanbridge/BridgeApplication;` (both also present in `out/app-classes.jar`, which was compiled correctly and WAS passed to d8).

Step 5 packaged only the first file:

```bash
(cd "$OUT/dex" && zip -q "$OUT/app-unsigned.apk" classes.dex)
```

`classes2.dex` was never added to the APK, so the shipped v0.2 contained zero app classes and crashed on launch with ClassNotFound for `BridgeApplication`/`MainActivity`.

**The fix (minimal, two lines in `build-manual.sh`):**
1. Step 5 now packages **every** dex file d8 produced: `(cd "$OUT/dex" && zip -q "$OUT/app-unsigned.apk" ./*.dex)` — with a comment in the script recording why.
2. Version bumped at the aapt2 link step: `--version-code 3 --version-name "0.2.1-v1-mock-fix"` so the fixed build installs over the broken one.

**Verification evidence (run against the final signed APK, mandatory after this failure):**
- APK dex listing: `classes.dex` (7,163,564) **and** `classes2.dex` (7,005,036) both present.
- Dex content greps (verbatim): `classes.dex` → no match for either app class (expected — libraries live there); `classes2.dex` → `Lcom/skynet/raybanbridge/MainActivity;` ✓ and `Lcom/skynet/raybanbridge/BridgeApplication;` ✓.
- `apksigner verify`: **PASS** (debug cert, SHA-256 `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b`).
- `aapt2 dump badging`: `package: name='com.skynet.raybanbridge' versionCode='3' versionName='0.2.1-v1-mock-fix'`, `application-label:'iNet'`, `launchable-activity: name='com.skynet.raybanbridge.MainActivity'`.
- Compiled manifest (`aapt dump xmltree`): meta-data `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓ (unchanged).
- Final APK: `manual-build/app-debug.apk`, 27,873,249 bytes (v0.2 was 25,530,255 — the delta is the previously-dropped second dex).

## HARDENING 2026-10-17 — v0.2.2-v1-mock-hardened (versionCode 4): startup-crash audit & survival pass

**Symptom:** v0.2.1 (both dexes packaged, app classes verified present) installed on Kenn's
Galaxy Z Fold7 (Android 17) and STILL crashed at launch — "iNet keeps stopping", no UI, no stack
trace available. The v1 code had never run anywhere; this pass audited every code path that
executes before the status screen can render, fixed what was concretely wrong, and made any
remaining failure visible on the phone screen itself.

### Audit findings, ranked by likelihood of being the launch crash

1. **Unguarded `Wearables.initialize()` in `Application.onCreate` — CONFIRMED structural
   defect (bytecode-read), prime suspect.** `BridgeApplication` called
   `Wearables.initialize(this)` and handled only the `DatResult` failure channel. Disassembly
   of `com.meta.wearable.dat.core.Wearables` (mwdat-core 1.0.1) shows `initialize()` has **no
   internal try/catch**: it calls `WearablesImpl.Companion.create(...)` and rethrows whatever
   escapes. `create()` synchronously runs, in order: `SoLoader.init(...)`,
   `DatConfiguration.fromContext(...)` (manifest meta-data parse),
   `Analytics.initialize(...)` (SQLite-backed), `CrashReporter.create/install`,
   `RegistrationManagerImpl.prepareWithContext(...)`, `ACDC.getInstance(...)` /
   `AirBridgeACDCProvider.getInstance(...)` (native library territory) and
   `DeviceManager.start(...)`. Any single throw in that chain — on a real, **unregistered**
   device with **no runtime permissions granted** — killed the process before the first frame,
   leaving exactly the observed symptom. Which sub-step throws on the Fold7 is unproven (no
   trace existed — this build exists to produce one).
2. **Missing merged manifest components from mwdat-core — CONFIRMED defect (manifest-read),
   plausible contributor.** The v1 report claimed "none of the five DAT AARs declare
   activities/services/providers/receivers" — **that claim was wrong for mwdat-core**.
   Its AAR manifest declares `<service android:name="com.meta.wearable.acdc.sdk.service.ACDCRegistrationService" android:exported="true">`
   (BIND intent-filter) and a `<queries>` block for `com.facebook.stella` /
   `com.facebook.stella_debug`. The manual pipeline does no manifest merging, so **neither was
   in the shipped APK**. Consequences: Android 11+ package-visibility filtering hides the Meta
   AI app from DAT, and the registration service the Meta AI app binds to in our process does
   not exist — both squarely in the init/registration path exercised at startup.
3. **Initialization ran with zero permissions granted, against Meta's own sample order —
   CONFIRMED by reading the SDK's samples/docs.** The CameraAccess sample requests
   BLUETOOTH/BLUETOOTH_CONNECT/INTERNET first and calls `Wearables.initialize` only **after
   the grants**; the DAT AGENTS.md display guidance likewise says "request runtime permissions
   before `Wearables.initialize(context)`". Our v1 called it at process start with nothing
   granted; if any init sub-step (e.g. `DeviceManager.start`) touches BluetoothAdapter APIs,
   that is a `SecurityException` on API 31+. Suspected mechanism, not proven.
4. **CLIENT_TOKEN placeholder ("PENDING_FROM_DEV_CENTER") — CLEARED as a launch suspect
   (bytecode-read).** `DatConfiguration.fromContext` reads APPLICATION_ID and CLIENT_TOKEN as
   plain strings via `getStringFromManifest`; there is no parsing or format validation at init
   time, so the placeholder is stored, not thrown on. It will matter later for
   production attestation/registration, not for process startup.
5. **Native library packaging — CLEARED (mechanical checks).** All 24 arm64 `.so` files are
   present; their `DT_NEEDED` closure is complete (only Android system libs —
   libandroid/libc/libdl/liblog/libm — are external); parsing the signed APK's local zip
   headers shows every `.so` entry stored uncompressed with its data offset aligned to both
   4096 and 16384 (zipalign `-p` survived signing), and ELF `PT_LOAD` alignment was verified
   16 KB in v1. No missing-lib or misalignment defect. `android:extractNativeLibs="true"` was
   still added (below) to remove the entire direct-mmap loading class from future suspicion.
6. **Theme/activity mismatch — CLEARED (by reading).** `MainActivity` is
   `androidx.activity.ComponentActivity` with the framework theme
   `@android:style/Theme.Material.NoActionBar`; there is no AppCompat/Material library in the
   dependency graph at all, so the classic AppCompat-theme crash cannot occur. UI is built
   programmatically (no layout inflation, no resource linking against library resources).
7. **Eager TTS construction on the path to first frame — SUSPECTED, low.** v1 constructed
   `AndroidTtsSpeaker` (which constructs `android.speech.tts.TextToSpeech`, binding to an
   external engine service) inside `onCreate`. Failures there are normally async, but it was
   an unguarded external dependency before first frame; now deferred and guarded (below).
8. **Eager foreground-service start — CLEARED for launch (by reading).** `BridgeService` is
   started only from `startStreaming()`, never at startup, so targetSdk-35 FGS start rules
   are not a launch factor. (Noted, out of scope for this pass: the `startForegroundService`
   call happens from a collector coroutine that could run while backgrounded; if that ever
   throws `ForegroundServiceStartNotAllowedException`, CrashGuard will now capture it.)

**Deviation check vs. DAT docs (toolchain `dat-android/AGENTS.md` + samples):** the docs show
`Wearables.initialize` either after permission grants in the Activity (CameraAccess,
DisplayAccess) or inside a repository with failure "logged and lived with" (BirdSpotter) —
no official sample initializes it bare in `Application.onCreate` the way v1 did. v0.2.2 moves
init behind a guarded, post-first-frame call, matching the samples' defensive posture.
Registration (`Wearables.startRegistration` + waiting for `RegistrationState.REGISTERED`
before session creation) remains unimplemented in the arm() flow, as in v1 — out of scope for
this startup pass, flagged for the next one.

### Code changes (startup survival only; gesture/state machine, stub endpoint, TTS stand-in,
local upload queue, and all feature scope unchanged)

- **NEW `crash/CrashGuard.kt`** — installs a default uncaught-exception handler that writes
  the full stack trace to `SkynetRayBanArchive/Miscellaneous/last-crash.txt` in the app's
  external files dir (internal-storage copy as fallback), stores a one-line summary in
  SharedPreferences with a synchronous `commit()`, best-effort pushes the line to a live UI
  sink, then **chains to the previously installed handler** (platform/DAT) so normal crash
  handling still runs. `recordError()` gives caught errors the same treatment. `install()` is
  re-assertable: DAT installs its own crash reporter during init, so DatInit re-installs the
  guard afterwards, wrapping DAT's handler as the chained "previous". Every guard method is
  internally try/catch-wrapped — the guard cannot become the crash.
- **NEW `DatInit.kt`** — the single guarded doorway into DAT. `ensureInitialized()` calls
  `Wearables.initialize` inside try/catch(Throwable); a typed `DatResult` failure, the
  `ALREADY_INITIALIZED` case (treated as usable), or a thrown exception are each recorded via
  CrashGuard and exposed as `lastInitStatus` for the UI. Never throws.
- **`BridgeApplication.kt`** — no longer touches DAT at all; installs CrashGuard (itself
  guarded) and nothing else.
- **`MainActivity.kt`** — UI build wrapped in try/catch with a last-ditch red error
  TextView fallback; new red **last-error line** under the status line, populated at startup
  from CrashGuard's stored summary (so a crash that killed the previous run is readable on
  the next launch) and updated live via `CrashGuard.liveErrorSink`. DAT init and TTS
  construction moved to `warmUp()`, posted after the first frame; TTS failure falls back to
  `LogOnlySpeaker`. All button flows (arm mock/real, disarm) run inside a `guarded()` helper
  that records + displays throws instead of crashing.
- **`bridge/BridgeController.kt`** — `arm()`/`disarm()` bodies wrapped (via internal impls)
  in try/catch(Throwable) with CrashGuard recording; `arm()` now calls
  `DatInit.ensureInitialized()` before `Wearables.createSession` and stays disarmed with a
  logged reason if init failed.
- **`AndroidManifest.xml`** — added the `<queries>` block for `com.facebook.stella` /
  `com.facebook.stella_debug` and the `ACDCRegistrationService` declaration copied verbatim
  from mwdat-core 1.0.1's AAR manifest (restoring what AGP merging would have supplied), and
  `android:extractNativeLibs="true"` on `<application>` so the native libs are extracted at
  install instead of mmap-loaded from the APK.
- **Version**: manual pipeline and Gradle `defaultConfig` both bumped to versionCode 4 /
  versionName `0.2.2-v1-mock-hardened` (installs over v0.2.1).

### Verification evidence (final signed APK, mandatory suite)

- APK: `manual-build/app-debug.apk`, 27,873,249 bytes — also copied to
  `~/workspace/your_files/iNet-hardened.apk`.
- Dex: `classes.dex` (7,163,564) and `classes2.dex` (7,019,676 — grew vs. v0.2.1's 7,005,036
  by the new guard code) both packaged. Greps: `classes2.dex` contains
  `Lcom/skynet/raybanbridge/MainActivity;` ✓, `Lcom/skynet/raybanbridge/BridgeApplication;` ✓,
  plus the new `Lcom/skynet/raybanbridge/crash/CrashGuard;` ✓ and
  `Lcom/skynet/raybanbridge/DatInit;` ✓ (classes.dex has none of them, as before — multidex
  layout unchanged).
- `apksigner verify`: **PASS** (same debug cert, SHA-256
  `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b`).
- `aapt2 dump badging`: `package: name='com.skynet.raybanbridge' versionCode='4'
  versionName='0.2.2-v1-mock-hardened'`, `application-label:'iNet'`,
  `launchable-activity: name='com.skynet.raybanbridge.MainActivity'`, targetSdkVersion 35.
- Compiled manifest (`aapt dump xmltree`): meta-data
  `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓; `<queries>` entries for
  `com.facebook.stella` and `com.facebook.stella_debug` ✓;
  `com.meta.wearable.acdc.sdk.service.ACDCRegistrationService` with its `.BIND` action ✓;
  `android:extractNativeLibs` = true ✓.

**Expected on-device behavior:** iNet should now reach the status screen no matter what DAT
does. If DAT init fails or throws on the Fold7, the screen shows it in red (and the log line
explains arming will retry/report), and any hard crash leaves its full stack trace in
`SkynetRayBanArchive/Miscellaneous/last-crash.txt` plus a red last-error line on the next
launch — the phone becomes the bug report.

## SHELL PROBE 2026-10-17 — v0.2.3-v1-shellprobe (versionCode 5): a launch path provably free of every library

**Symptom that forced this build:** v0.2.2 (startup-hardened, CrashGuard installed from
Application.onCreate, DAT init deferred until after first frame) STILL crashed on launch on
Kenn's Fold7 — no screen ever rendered, no error ever displayed, despite CrashGuard being
designed to surface exactly that. Verified before this build: both dexes ship, ALL app
classes and the DAT classes (incl. MockDeviceKit/MockGlasses, 2,188 com/meta/wearable
descriptors) are present in the APK, signature/manifest fine.

**Working theory:** the failure happens during class loading/verification of MainActivity
itself. The old MainActivity's fields/imports put DAT types in its resolution graph (a
`MockGlasses` field, `CapturePressType`, `BridgeController`, plus AndroidX
ComponentActivity/lifecycle and kotlinx.coroutines). If anything in that chain fails to
load/link/verify on the real device, MainActivity can never load — and CrashGuard's stored
error (written to SharedPreferences by the Application-installed handler) can never be
DISPLAYED, because the displaying class is the dead one. The handler may well have been
recording the culprit all along, into a screen nobody could reach.

### Design (startup survival is the entire point; no features added)

- **`MainActivity` REPLACED with a shell**: a plain `android.app.Activity` (framework
  Activity — NOT ComponentActivity; no AndroidX, no Compose, no coroutines). No field,
  parameter, return type, import, or signature references any type outside `android.*`,
  `java.*`, `kotlin.*`, and the app's own crash package (`CrashGuard`, itself
  android/java-only). UI is built in code (LinearLayout + TextViews + Buttons, framework
  widgets only) and shows, in order: "iNet" title, "Skynet's eyes and ears" subtitle,
  "Build: 0.2.3-v1-shellprobe (versionCode 5)", state line starting
  "State: SHELL — engine not loaded", the CrashGuard stored-error line in red (read at
  startup — this may finally surface the v0.2.2 crash the old MainActivity could never
  display; CrashGuard and its SharedPreferences/file storage are byte-compatible, keys
  unchanged, so the error recorded by previous builds is readable), a scrolling log area,
  and TWO probe buttons.
- **QUARANTINE facades** in the new subpackage `com.skynet.raybanbridge.probe`:
  `DatEngineProbe` is the only new class allowed to reference DAT core types
  (`Wearables`, `DeviceSession`, `DatResult`); `MockKitProbe` is the only one allowed to
  reference mockdevice types (`MockDeviceKit`, `MockGlasses`, `GlassesModel`). Each has a
  static (`@JvmStatic`) `probe(): String` that performs CLASS LOADING ONLY — it resolves
  the class literals and Class.forName-initializes each target (running static
  initializers) and returns the loaded class names. NO `Wearables.initialize`, no session,
  no Context, no permissions anywhere in either probe.
- **MainActivity references the facades ONLY by fully-qualified name string** via
  `Class.forName(...)` + reflective `Method.invoke` of `probe()` — zero compile-time
  references. Each button handler wraps the whole probe in try/catch(Throwable); on
  failure it appends the FULL exception chain (class + message + cause chain) and the
  complete stack trace to the on-screen log, shows the one-liner in the red error line,
  and records it via CrashGuard. Loading a class runs its static initializers — that is
  precisely the diagnostic: if a `<clinit>` or resolution chain explodes, the exception
  text appears on a live screen.
- **The old MainActivity is now `LegacyMainActivity`** (class renamed, kept compiling,
  NOT in the manifest, never launched) — the full v1 status screen survives as the
  reference implementation for the build that restores the real UI after this diagnosis.
  All other code (BridgeController, conversation, archive, DatInit) is unchanged and
  unreachable from the launch path. `Wearables.initialize` is called NOWHERE in this
  build's reachable paths (launch path: BridgeApplication → CrashGuard only; shell:
  framework widgets + CrashGuard reads only).
- **`BridgeApplication` kept as-is** (CrashGuard only) — and now verified at the compiled
  class level (below) to reference no com/meta/wearable types.
- **Version**: versionCode 5 / versionName `0.2.3-v1-shellprobe` in both the manual
  pipeline and the Gradle defaultConfig; package, label, and icon identical — installs as
  an update over v0.2.2 (same debug cert).

### Class-graph gate evidence (the mandatory pre-delivery proof)

Occurrence counts of banned substrings in the PRE-DEX compiled classes extracted from
`manual-build/out/app-classes.jar` (grep -a -o | wc -l):

| class | "meta/wearable" | "androidx" | "kotlinx" |
|---|---|---|---|
| `com/skynet/raybanbridge/MainActivity.class` | **0** | **0** | **0** |
| `com/skynet/raybanbridge/BridgeApplication.class` | **0** | **0** | **0** |

Positive controls (the gate is not vacuously passing): the same MainActivity.class DOES
contain `android/app/Activity` (2 occurrences), `crash/CrashGuard` (2), and the probe
facade name string `probe.DatEngineProbe` (1, a plain string literal — the only way the
shell knows the facade). The quarantined facades carry the DAT references instead:
`probe/DatEngineProbe.class` and `probe/MockKitProbe.class` each contain 3
"meta/wearable" occurrences, exactly as designed.

### APK verification (final signed APK)

- APK: `manual-build/app-debug.apk`, 27,889,633 bytes — also copied to
  `~/workspace/your_files/iNet-shellprobe.apk` (md5 `b71a24be5fd66ceb9397214bbcfc15d4`,
  identical for both copies).
- `apksigner verify`: **PASS** (same debug cert as v0.2.1/v0.2.2).
- `aapt2 dump badging`: `package: name='com.skynet.raybanbridge' versionCode='5'
  versionName='0.2.3-v1-shellprobe'`, `application-label:'iNet'`,
  `launchable-activity: name='com.skynet.raybanbridge.MainActivity'`, targetSdkVersion 35.
- Dex: `classes.dex` (7,163,564) and `classes2.dex` (7,029,864) both packaged.
  `classes2.dex` contains `Lcom/skynet/raybanbridge/MainActivity;` ✓,
  `Lcom/skynet/raybanbridge/BridgeApplication;` ✓,
  `Lcom/skynet/raybanbridge/probe/DatEngineProbe;` ✓,
  `Lcom/skynet/raybanbridge/probe/MockKitProbe;` ✓, and
  `Lcom/skynet/raybanbridge/LegacyMainActivity;` ✓ (present, compiling, unlaunched).
- Compiled manifest (`aapt dump xmltree`): meta-data
  `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓; `<queries>` entries
  for `com.facebook.stella` / `com.facebook.stella_debug` ✓;
  `com.meta.wearable.acdc.sdk.service.ACDCRegistrationService` with its `.BIND` action ✓
  (the v0.2.2 manifest restorations all still present).

### What each on-screen outcome will mean

- **Shell won't open at all** → the problem is BELOW the activity class graph: package /
  install / Application-level (BridgeApplication + CrashGuard are the only app code that
  runs before the shell, and both are now proven library-free at the class level) — or a
  native-level failure in process startup. The hunt leaves the DAT class graph entirely.
- **Shell opens and the red line shows a stored error** → that text is the v0.2.2 (or
  earlier) launch killer, recorded by CrashGuard's Application-installed handler and
  finally readable — the culprit is named before any button is pressed.
- **A probe button fails** → the log area shows the failing chain's full exception text
  (which class, which initializer, caused by what). "Test: load glasses engine" failing
  indicts the DAT core graph; "Test: load mock kit" failing indicts the mockdevice graph
  (the old MainActivity's `MockGlasses` field made that graph launch-critical).
- **Both probes succeed** → the DAT class graph (loading + static init) is healthy on the
  real device, and the hunt moves to RUNTIME initialization: `Wearables.initialize`'s
  synchronous work (SoLoader/config/analytics/registration/ACDC) and the registration
  flow — exercised next, behind the same guarded, on-screen pattern.

## SHELL PROBE RESULTS — 2026-10-22 07:20 ET (Kenn's Fold7, real hardware)

Kenn ran v0.2.3-v1-shellprobe and sent the screenshot (his two-taps gate, owed since Oct 17 — closed).

- **TEST: LOAD GLASSES ENGINE → PROBE OK** (tapped twice, same result both times): DAT core classes load + initialize OK — `com.meta.wearable.dat.core.Wearables`, `com.meta.wearable.dat.core.session.DeviceSession`, `com.meta.wearable.dat.core.types.DatResult`.
- **TEST: LOAD MOCK KIT → PROBE OK**: Mock kit classes load + initialize OK — `com.meta.wearable.dat.mockdevice.MockDeviceKit`, `com.meta.wearable.dat.mockdevice.api.MockGlasses`, `com.meta.wearable.dat.mockdevice.api.GlassesModel`.
- **Last error: (none recorded)** — CrashGuard held nothing from prior builds either.

**Verdict:** both quarantined class graphs (DAT core + mockdevice) load and static-init cleanly on the real device. The v0.2.2 launch crash was NOT in class loading or static initialization. Per this report's decision tree, the hunt moves to **RUNTIME initialization**: `Wearables.initialize`'s synchronous work (SoLoader / config / analytics / registration / ACDC) and the registration flow — to be exercised next behind the same guarded, on-screen probe pattern (v0.2.4 runtime-init probe).

## RUNTIME-INIT PROBE — v0.2.4-v1-runtimeprobe (versionCode 6): Wearables.initialize, exercised on a live screen

**Built 2026-10-22** (after the Oct 22 shell-probe results above: both class graphs PROBE OK on the real Fold7, class loading/static init exonerated). Same manual pipeline (`manual-build/build-manual.sh`), same package, label "iNet", icon, and debug cert as v0.2.3 — installs as an in-place update over v0.2.3.

### What was built

The shell is unchanged in kind: `MainActivity` still references NOTHING outside `android.*`, `java.*`, `kotlin.*` and the app's own crash package, and both v0.2.3 probe buttons work exactly as before ("Test: load glasses engine", "Test: load mock kit"). The build line now reads "Build: 0.2.4-v1-runtimeprobe (versionCode 6)". One button added:

- **"Test: initialize runtime"** → loads **`com.skynet.raybanbridge.probe.RuntimeInitProbe`** by fully-qualified name string and reflectively invokes its static `probe(Context)` with the application context — the same Class.forName + `Method.invoke` pattern as the existing facades (the shell now also imports `android.content.Context` to name the signature; Context is `android.*`, shell-legal). Failure handling is byte-for-byte the existing handler shape: full exception chain + complete stack trace into the on-screen log, one-liner in the red error line, CrashGuard record. On success the state line reads "State: SHELL — runtime INITIALIZED by probe (no session created)".

### The facade

`probe/RuntimeInitProbe.kt` is the only new class allowed to touch the runtime init path. It:

1. Takes the application context (`context.applicationContext`) — passed in reflectively, no compile-time link from the shell.
2. Calls the SDK's real initialization entry point, **`Wearables.initialize(appContext)`** — the identical call `DatInit.ensureInitialized` makes (DatInit is v0.2.2's guarded doorway and remains the production path), but deliberately **without DatInit's catch-all**: a throw must reach the shell's probe handler raw so the full chain prints on screen. Result handling mirrors DatInit's semantics step for step: success → report; `WearablesError.ALREADY_INITIALIZED` → treated as initialized (second tap is safe); any other typed `DatResult` failure → thrown as an `IllegalStateException` carrying the error's name + `description`, with the SDK's `onFailure` cause chained in, so it lands in the same on-screen failure path. After initialize returns, CrashGuard is re-asserted (DAT installs its own crash reporter during init; same `finally`-equivalent as DatInit).
3. Reports post-init state through the SDK's **read-only** APIs — verified present in mwdat-core 1.0.1 by `javap` before use: `Wearables.registrationState` (`StateFlow<RegistrationState>`: UNAVAILABLE / AVAILABLE / REGISTERING / REGISTERED / UNREGISTERING), `Wearables.devices` (`StateFlow<Set<DeviceIdentifier>>`, count only), and the `Wearables.isDevMode` property. Each read is individually guarded — a throwing read reports itself "unavailable (…)" instead of masking the initialize verdict. There is **no global session-state API** (sessions are created objects; `getSessionManager` is internal-mangled), and this probe creates no session — the report text says exactly that rather than guessing. Registration is read, never driven (`startRegistration` needs an Activity for Meta's flow; that is a later build).

One compile fix during the build, recorded honestly: Kotlin exposes the SDK's `isDevMode()` as a property, so the first compile failed on `Wearables.isDevMode()` ("cannot be invoked as a function"); the facade now reads `Wearables.isDevMode`. No other changes were needed.

### Gate evidence (mandatory suite, run against the final signed APK)

- **Dex payload greps** — both dexes extracted from the final APK (`classes.dex` 7,163,564, unchanged; `classes2.dex` 7,034,740, grew from v0.2.3's 7,029,864 by the new code). `classes2.dex` contains ALL of: `Lcom/skynet/raybanbridge/MainActivity;` ✓, `Lcom/skynet/raybanbridge/BridgeApplication;` ✓, `Lcom/skynet/raybanbridge/probe/DatEngineProbe;` ✓, `Lcom/skynet/raybanbridge/probe/MockKitProbe;` ✓, and the new `Lcom/skynet/raybanbridge/probe/RuntimeInitProbe;` ✓.
- **Class-graph gate** (pre-dex `out/app-classes.jar`, same method as v0.2.3): `MainActivity.class` — "meta/wearable" **0**, "androidx" **0**, "kotlinx" **0**; `BridgeApplication.class` — 0/0/0. Positive controls: MainActivity.class contains `android/app/Activity` (2) and exactly one occurrence each of the plain-string facade names `probe.DatEngineProbe`, `probe.MockKitProbe`, `probe.RuntimeInitProbe` — the only way the shell knows the facades. The facades carry the DAT references instead: DatEngineProbe 3, MockKitProbe 3, **RuntimeInitProbe 12** "meta/wearable" occurrences (initialize + state reads), exactly as designed.
- **`apksigner verify`: PASS** — signer cert SHA-256 `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b`, identical to v0.2.1/v0.2.2/v0.2.3 (in-place update preserved).
- **`aapt2 dump badging`**: `package: name='com.skynet.raybanbridge' versionCode='6' versionName='0.2.4-v1-runtimeprobe'`, `application-label:'iNet'`, targetSdkVersion 35.
- **Compiled manifest** (`aapt2 dump xmltree`): meta-data `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓ (CLIENT_TOKEN meta-data also still present; `<queries>` + ACDCRegistrationService restorations from v0.2.2 unchanged).
- **Final APK**: `manual-build/app-debug.apk`, **27,889,633 bytes**, **sha256 `3a43f93bc18e5ac3ed558669ef645a473d15b5668eadc838f31ecf4f20a0f540`** (md5 `7cf4ec82548a1c82496758bd475be46c` — differs from v0.2.3's `b71a24be…`; the byte SIZE is coincidentally identical to v0.2.3 because zipalign padding absorbed the small dex growth, the hashes prove it is the new artifact). Copy at `~/workspace/your_files/iNet-runtimeprobe.apk` (sha256 verified identical after copy). Version also bumped in the Gradle `defaultConfig` (versionCode 6 / `0.2.4-v1-runtimeprobe`) for parity, as with every prior build.

### What Kenn's on-phone run will mean (result-decision tree)

- **PROBE OK + "Registration state: REGISTERED"** → the entire synchronous init chain (SoLoader → config parse → analytics → crash reporter → registration prepare → ACDC → DeviceManager) completes on the real, registered device. Runtime init is exonerated too, and the v0.2.2 launch killer lives LATER: the old MainActivity's own startup work (TTS construction, warm-up ordering) or the session-creation path. Next probe layer: session creation behind the same pattern.
- **PROBE OK + "Registration state: AVAILABLE" (or UNAVAILABLE)** → init completes, but the glasses are NOT registered to this app. That is itself a headline finding: every prior build armed toward `createSession` on an unregistered device. Next build is the registration flow itself (`Wearables.startRegistration` + intent return, behind the guarded pattern) — a feature step, not another diagnostic.
- **PROBE FAILED with a thrown exception** → the stack trace in the log names the exact `create()` sub-step that explodes (frames will show SoLoader / DatConfiguration / Analytics / RegistrationManagerImpl / ACDC / DeviceManager). **That sub-step is the v0.2.2 launch killer, finally named** — the fix then targets it specifically (permissions-before-init ordering per Meta's samples if it's DeviceManager/Bluetooth, CLIENT_TOKEN or manifest components if it's config/registration, native-load handling if it's SoLoader/ACDC).
- **PROBE FAILED with a typed failure** (log shows `IllegalStateException: Wearables.initialize returned a typed failure: …`) → initialize refused cleanly rather than exploding; the WearablesError name + description in the text says why, and the hunt follows that error, not a crash.
- **Shell won't open at all** → unchanged from the v0.2.3 tree: the problem is below the activity class graph (the shell's graph is proven identical to v0.2.3 plus one string literal and one `android.content` import), and the hunt leaves DAT entirely.

## RUNTIME-INIT PROBE RESULTS — 2026-10-22 07:30 ET (Kenn's Fold7)

Kenn installed v0.2.4 over v0.2.3 and ran all three buttons (screenshot filed by Kenn in chat).

- **TEST: LOAD GLASSES ENGINE → PROBE OK** (again).
- **TEST: LOAD MOCK KIT → PROBE OK** (again).
- **TEST: INITIALIZE RUNTIME → PROBE FAILED — and the failure is the catch of the whole probe program:**

```
java.lang.NoSuchMethodError: No static method runBlockingK(Lkotlinx/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;)Ljava/lang/Object; in class Lkotlinx/coroutines/BuildersKt; or its super classes
  at com.meta.wearable.acdc.sdk.store.ACDCStore$Companion.getInstance
  at com.meta.wearable.acdc.sdk.registrar.ACDCSecureRegistrarImpl.<init>
  at com.meta.wearable.acdc.sdk.registrar.ACDCSecureRegistrarImpl.<init>
  at com.meta.wearable.dat.core.internal.registration.RegistrationManagerImpl.prepareWithContext
  at com.meta.wearable.dat.core.internal.WearablesImpl$Companion.create
  at com.meta.wearable.dat.core.internal.WearablesImpl$Companion.create$default
  at com.meta.wearable.dat.core.Wearables.initialize
  at com.skynet.raybanbridge.probe.RuntimeInitProbe.probe(RuntimeInitProbe.kt:60)
```

**Diagnosis:** `Wearables.initialize` itself RUNS — it walks create() → prepareWithContext → builds the ACDC secure registrar → asks ACDCStore for its instance, which calls `BuildersKt.runBlocking(CoroutineContext, Function2)` — an overload the kotlinx-coroutines library packaged by OUR manual pipeline does not carry. This is a **coroutines version skew between the DAT SDK (mwdat-core 1.0.1, compiled against a newer coroutines) and the coroutines artifact our build bundled**. Not a Meta wall, not a permissions/account block, not the glasses: a dependency-alignment defect in our own packaging. It is also the prime suspect for the v0.2.2 launch death — the same initialize call through the same doorway, unguarded at startup, dying on an Error before any screen could render.

**Fix (next build, v0.2.5):** determine the kotlinx-coroutines version mwdat-core 1.0.1 declares (its POM/module metadata) and package exactly that coroutines artifact in the manual pipeline; rebuild the runtime probe unchanged otherwise; Kenn re-runs the same button. Expected outcome if the diagnosis is right: initialize completes and the probe reports registrationState + device count + isDevMode.

## COROUTINES-FIX PROBE — v0.2.5-v1-coroutinesfix (versionCode 7): the SDK's declared coroutines, packaged at last

**Built 2026-10-22** (after the Oct 22 runtime-init results above: initialize dies inside ACDCStore on a missing `BuildersKt.runBlockingK`). Same manual pipeline (`manual-build/build-manual.sh`, log `build-coroutinesfix-run.txt`), same package, label "iNet", icon, and debug cert as v0.2.3/v0.2.4 — installs as an in-place update over v0.2.4. The probe itself is byte-for-byte the v0.2.4 instrument: same shell, same three buttons, same facades (`RuntimeInitProbe` logic untouched — it still carries exactly 12 "meta/wearable" references, same as v0.2.4). **The only functional change is the kotlinx-coroutines version on the dex classpath: 1.6.4 → 1.11.0.**

### Investigation: bundled vs declared vs packaged

**(a) What the pipeline bundled (v0.2.4 and every prior build).** `manual-build/resolved.txt` — the manual resolver's frozen output and the build script's only input list — pinned `kotlinx-coroutines-android:1.6.4`, `kotlinx-coroutines-core-jvm:1.6.4`, and `kotlinx-coroutines-core:1.7.3`; the m2 store held exactly those jars. The `BuildersKt` that reached the dex came from **core-jvm 1.6.4**. Verified by `javap`: 1.6.4's `BuildersKt` declares `runBlocking(CoroutineContext, Function2)` and `runBlocking$default` — but **no `runBlockingK`** (that method does not exist in the 1.6.4 line). The `kotlinx-coroutines-core:1.7.3` jar is a classless multiplatform redirect (0 `.class` files — verified by listing), so it never supplied a competing `BuildersKt`.

**(b) What the caller actually invokes.** Extracted from the shipped SDK itself (`mwdat-core-1.0.1.aar` → `classes.jar`, `javap -c` on `com/meta/wearable/acdc/sdk/store/ACDCStore$Companion`): the bytecode contains `invokestatic kotlinx/coroutines/BuildersKt.runBlockingK:(Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;)Ljava/lang/Object;` — descriptor-identical to the NoSuchMethodError Kenn's phone threw. The SDK was compiled against a coroutines whose `BuildersKt` grew a `runBlockingK` entry point; ours predates it.

**(c) What the SDK declares.** `local-repo/com/meta/wearable/mwdat-core/1.0.1/mwdat-core-1.0.1.pom` (the only mwdat POM naming coroutines) declares `org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0`. The project's own Gradle catalog agrees: `gradle/libs.versions.toml` pins `coroutines = "1.11.0"`. **The SDK-declared version is 1.11.0** — determined from local artifacts, not the web. Verified before packaging: `javap` on the fetched `kotlinx-coroutines-core-jvm-1.11.0.jar` shows `BuildersKt` declaring `public static final <T> T runBlockingK(kotlin.coroutines.CoroutineContext, kotlin.jvm.functions.Function2...)` alongside `runBlocking`, `runBlocking$default`, `runBlockingImpl`, and `runBlockingK$default`. The 1.11.0 POM graph is self-consistent with what we already bundle: coroutines-android 1.11.0 → core-jvm 1.11.0 + kotlin-stdlib 2.2.20 (we dex stdlib 2.2.0 / compile with 2.2.21 — same 2.2 line, metadata-compatible); core-jvm 1.11.0 → org.jetbrains:annotations 23.0.0 (compile-time-only annotations; the pipeline's existing annotations 13.0 jar stays, unchanged).

**(d) How 1.6.4 got in — the mechanism.** The manual resolver (`resolve_deps.py`) is **first-seen-wins**: its `seen` map records the first version encountered for a coordinate and never upgrades it — the opposite of Gradle's newest-wins. The local POM graph declares coroutines three ways: **1.6.4** from the androidx lifecycle 2.6.x POMs (lifecycle-common 2.6.2, lifecycle-runtime-ktx 2.6.1, lifecycle-viewmodel-ktx 2.6.1, lifecycle-viewmodel-savedstate 2.6.1 — all verified in m2), **1.7.3** from androidx.activity 1.10.1's POM, and **1.11.0** from mwdat-core. Under first-seen-wins the older declarations stuck; a real Gradle build (catalog pin + newest-wins) would have resolved 1.11.0 all along. Our packaging defect, fully explained: a hand-rolled resolver quietly outvoted both Meta's POM and our own version catalog.

### What changed

- `manual-build/m2/`: fetched from Maven Central into the local store — `kotlinx-coroutines-android/1.11.0` (jar 17,819 B + POM), `kotlinx-coroutines-core-jvm/1.11.0` (jar 1,577,052 B + POM), `kotlinx-coroutines-core/1.11.0` (jar 145,895 B, 0 `.class` files — the same classless redirect shape as the 1.7.3 it replaces + POM).
- `manual-build/resolved.txt`: the three coroutines lines now read 1.11.0 / 1.11.0 / 1.11.0. Nothing else in the list moved (51 classpath jars before and after — a pure 1:1 coordinate swap).
- Versions bumped in the three customary places: `build-manual.sh` aapt2 link (`--version-code 7 --version-name "0.2.5-v1-coroutinesfix"`), Gradle `defaultConfig` (versionCode 7 / `0.2.5-v1-coroutinesfix`), and the shell's build line in `MainActivity.kt` ("Build: 0.2.5-v1-coroutinesfix (versionCode 7)").
- Nothing else: no source logic changes, no other dependency changes, the EXCLUDE_REGEX untouched (its `kotlinx-coroutines-core/1.7.3` alternative simply no longer names a resolved artifact; it never matched a filename anyway — the dash/slash mismatch documented implicitly by every prior build's surviving extracted jar — and the classless core jar is inert in the dex either way).

### Gate evidence (mandatory suite, run against the final signed APK)

- **NEW GATE — the fix itself, proven in the artifact (dexdump, method level).** Both dexes extracted from the final APK; `dexdump` locates the **definition** of `Lkotlinx/coroutines/BuildersKt;` in `classes2.dex`, and its method table reads:
  - `runBlocking : (Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;)Ljava/lang/Object;`
  - `runBlocking$default : (Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;ILjava/lang/Object;)Ljava/lang/Object;`
  - `runBlockingImpl : (Lkotlin/coroutines/CoroutineContext;Lkotlinx/coroutines/EventLoop;Lkotlin/jvm/functions/Function2;)Ljava/lang/Object;`
  - **`runBlockingK : (Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;)Ljava/lang/Object;`** ✓ — the exact method, with the exact descriptor, whose absence threw on Oct 22
  - `runBlockingK$default : (Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;ILjava/lang/Object;)Ljava/lang/Object;`
  This is a full method-table proof from the shipped dex, not a string-level inference.
- **Dex payload greps** — `classes.dex` 7,165,444 B and `classes2.dex` 7,187,752 B (both grew/shifted vs v0.2.4's 7,163,564 / 7,034,740: the 1.11.0 core is a larger library and d8 re-split the multidex layout). `classes2.dex` contains ALL of: `Lcom/skynet/raybanbridge/MainActivity;` ✓, `Lcom/skynet/raybanbridge/BridgeApplication;` ✓, `Lcom/skynet/raybanbridge/probe/DatEngineProbe;` ✓, `Lcom/skynet/raybanbridge/probe/MockKitProbe;` ✓, `Lcom/skynet/raybanbridge/probe/RuntimeInitProbe;` ✓.
- **Class-graph gate** (pre-dex `out/app-classes.jar`, same method as v0.2.3/v0.2.4): `MainActivity.class` — "meta/wearable" **0**, "androidx" **0**, "kotlinx" **0**; `BridgeApplication.class` — 0/0/0. Positive controls: exactly one occurrence each of `probe.DatEngineProbe`, `probe.MockKitProbe`, `probe.RuntimeInitProbe` in MainActivity.class. The shell's purity is unchanged — the coroutines swap is invisible to it, exactly as designed (and note the irony the gate now certifies: the shell still references zero `kotlinx` classes while the library *behind* the facade wall is what changed).
- **`apksigner verify`: PASS** — signer cert SHA-256 `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b`, identical to v0.2.1–v0.2.4 (in-place update preserved).
- **`aapt2 dump badging`**: `package: name='com.skynet.raybanbridge' versionCode='7' versionName='0.2.5-v1-coroutinesfix'`, `application-label:'iNet'`, targetSdkVersion 35.
- **Compiled manifest** (`aapt dump xmltree`): meta-data `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓ (CLIENT_TOKEN meta-data, `<queries>`, and ACDCRegistrationService restorations unchanged, as in v0.2.4).
- **Final APK**: `manual-build/app-debug.apk`, **27,938,785 bytes** (grew from v0.2.4's 27,889,633 by the larger coroutines core), **sha256 `4f92574af702cdfabd9f653091766c45977d7e9837d67cff7eda5e4c27876abb`**, md5 `01f625d2146edc76835c7fd45909a984`. Copy at `~/workspace/your_files/iNet-coroutinesfix.apk` — sha256 verified identical after copy.
- Build hygiene note: d8 warnings 8,487 vs v0.2.4's 6,737 — same notice classes as before (the "malformed kotlin.Metadata 2.2.0 vs max supported 2.1.0" Info lines predate this change; stdlib 2.2.0 was already bundled), the count tracks the larger 1.11.0 graph. Warnings, not errors; the build completed, signed, and verified.

### What Kenn's re-run will mean (result-decision tree)

Kenn installs v0.2.5 over v0.2.4 and taps the same third button, **"Test: initialize runtime"**.

- **PROBE OK + "Registration state: REGISTERED"** → the coroutines diagnosis is confirmed end-to-end AND the whole synchronous init chain (SoLoader → config → registration prepare → ACDC store → DeviceManager) completes on the real, registered device. Two mysteries close at once: the Oct 22 failure and (by the chain of custody — same call, same doorway, unguarded at startup) the v0.2.2 launch death. The hunt moves past init entirely: next probe layer is session creation, behind the same guarded pattern.
- **PROBE OK + "Registration state: AVAILABLE" (or UNAVAILABLE)** → the diagnosis is still confirmed (initialize completed — the runBlockingK wall is gone), but the glasses are not registered to this app. Next build is the registration flow itself (`Wearables.startRegistration` + intent return, guarded) — a feature step, not another diagnostic.
- **PROBE FAILED with a NEW NoSuchMethodError / NoClassDefFoundError naming a different kotlinx class or method** → the skew is deeper than one method: some other part of the SDK graph was compiled against a coroutines surface our 1.11.0 still doesn't match (or another library in the graph has its own version skew of the same first-seen-wins kind). The failing descriptor names the next alignment target; the fix pattern is now proven — read the caller's POM, package its declared version. Also worth auditing the rest of resolved.txt for the same disease (notably tink: mwdat declares 1.20.0, we bundle 1.8.0 — flagged here as a *suspect to check if the next failure points at crypto*, not a proven defect).
- **PROBE FAILED with the SAME runBlockingK error** → the diagnosis was wrong about which artifact supplies BuildersKt at runtime (e.g. another jar shadowing the class earlier in the dex order). The dexdump proof above makes this unlikely — the defining class in the shipped classes2.dex carries the method — but if it happens, the hunt turns to dex class-ordering/shadowing, not versions.
- **PROBE FAILED with a thrown exception from a deeper create() sub-step** (SoLoader / config / ACDC native / DeviceManager frames) → init got *past* the coroutines wall and died at the next real obstacle; that sub-step is the new named target, exactly as the v0.2.4 tree described. Progress, not regression: each probe run retires one layer.
- **PROBE FAILED with a typed failure** (`IllegalStateException: Wearables.initialize returned a typed failure: …`) → initialize refused cleanly; the WearablesError name + description says why, and the hunt follows that error.
- **Shell won't open at all** → unchanged from prior trees: below the activity class graph; the hunt leaves DAT entirely.

## RUNTIME-INIT PROBE RESULTS — v0.2.5 (2026-10-22, Kenn's screenshot 07:49 ET)

Kenn installed v0.2.5 over v0.2.4 on the Fold7 and ran all three probes. Screenshot filed from his phone (chat upload 36628_284_huhp.jpg).

- **Glasses engine: PROBE OK** (DAT core classes load + initialize, as before).
- **Mock kit: PROBE OK** (as before).
- **Runtime init: PROBE OK — `Wearables.initialize` returned SUCCESS.** The full synchronous init chain (SoLoader / config / analytics / crash reporter / registration prepare / ACDC / device manager) completed without throwing. **The coroutines diagnosis is CONFIRMED end-to-end**: packaging the SDK-declared kotlinx-coroutines 1.11.0 retired the `runBlockingK` NoSuchMethodError. By chain of custody (same call, unguarded at startup), this also closes the v0.2.2 launch-death mystery.
- **Registration state: UNAVAILABLE. Dev mode: false. Known devices: 0.** Per the v0.2.5 decision tree: initialize is fixed, but the glasses are not registered to this app. The red "Last error" line on screen is the persisted 07:30 v0.2.4 error, not a new failure.
- **State line reads: "SHELL — runtime INITIALIZED by probe (no session created)"** — expected; this probe creates no session by design.

**Next step (per the tree): the registration flow build** — a guarded `Wearables.startRegistration` step (+ intent return handling), then re-probe registration state. Feature step, not another diagnostic. Awaiting Kenn's step-approval.

## REGISTRATION FLOW — v0.2.6-v1-registration (versionCode 8): the guarded startRegistration probe

**Built 2026-10-22** (Kenn approved the step after the v0.2.5 results above: initialize SUCCESS, registration UNAVAILABLE). Same manual pipeline (`manual-build/build-manual.sh`, log `build-registration-run.txt`), same package, label "iNet", icon, and debug cert as v0.2.3–v0.2.5 — installs as an in-place update over v0.2.5. Buttons 1–3 are unchanged in behavior; **one button added: "Test: start registration"**.

### SDK API evidence (researched BEFORE building — from the shipped mwdat-core 1.0.1 AAR via javap, cross-checked against Meta's AGENTS.md / permissions-registration skill in the toolchain DAT repo; no guessed APIs)

- **Entry point: `Wearables.startRegistration(android.app.Activity)` returns `void`** — fire-and-observe, NOT a DatResult. It delegates (under a lock) to `RegistrationManagerImpl.askRegistration(activity, register=true)`, which reads `DatConfiguration.fromContext` (our manifest APPLICATION_ID + app label), mints a `linkingFlowId` UUID, checks the current state (e.g. REGISTERED ⇒ emits `RegistrationError.ALREADY_REGISTERED`), then builds and launches the handoff intent. `startUnregistration(Activity)` is the mirror.
- **The handoff intent** (`IntentLauncherImpl.buildRegistrationIntent`, bytecode-read): `Intent(ACTION_VIEW, fb-viewapp://stella/dat/registration?appPackage=<our package>&appName=<label>&action=register&metaAppId=<APPLICATION_ID>&appPublicKeySha=<signing cert SHA>&minMWAVersion=292&attestationSessionId=…&attestationCertChain=…&linkingFlowId=<uuid>[&toolsEnabled=true])`, flag `0x10000` added. Before launching, `PackageManager.resolveActivity` is checked — unresolvable ⇒ the flow fails with `RegistrationError.META_AI_NOT_INSTALLED` (surfaced via `setRegistration(errorState, error)` onto the error stream).
- **Launch + result mechanics** (`RegistrationManagerImpl.launchIntentForResult`, bytecode-read): if the host Activity is an `androidx.activity.ComponentActivity`, the SDK registers an `ActivityResultLauncher` on its registry (key `"dat_registration_result"`, StartActivityForResult contract); its callback maps RESULT_CANCELED while REGISTERING ⇒ `FAILED_TO_REGISTER` ("Registration activity was cancelled by user"), and otherwise reads the result-data extra **`"is_dev_mode"`** and unregisters itself. **If the host is a PLAIN Activity, the SDK falls back to `activity.startActivityForResult(intent, 0)` and registers NO callback** — the result returns to the host's own `onActivityResult` (requestCode 0) and the SDK never sees it. Our shell is deliberately a plain Activity (the v0.2.3 quarantine), so v0.2.6's shell captures the result there and forwards it to the facade for reporting.
- **Completion does NOT ride the activity result.** The Meta AI app completes a secure handshake by binding to the `ACDCRegistrationService` declared in OUR manifest (the v0.2.2 restoration); `RegistrationManagerImpl.setupRegistrarMonitoring` observes the ACDC secure registrar and calls `setRegistration(REGISTERED, null)` (telemetry `flowDidSucceed`) when it reports registered. The verdict surface is therefore **`Wearables.registrationState: StateFlow<RegistrationState>`** — UNAVAILABLE / AVAILABLE / REGISTERING / REGISTERED / UNREGISTERING (bytecode-verified enum) — sampled live by the facade after any return.
- **Errors: `Wearables.registrationErrorStream: Flow<RegistrationError>`** (SharedFlow on the impl — emissions with no collector are lost, which is why the facade arms its collectors BEFORE calling startRegistration). `RegistrationError` enum (bytecode-verified): ALREADY_REGISTERED, ALREADY_UNREGISTERED, FAILED_TO_REGISTER, FAILED_TO_UNREGISTER, META_AI_NOT_INSTALLED, UNKNOWN — each a `DatError` with a `description`, matching the established reporting format.
- **`Wearables.handleIntent(Intent, Function1<RegistrationRequest, Unit>): DatResult<Boolean, RegistrationRequestError>` is a SEPARATE path** — registration requests initiated from inside the Meta AI app. `MWARegistrationIntentHandler` recognizes intents carrying extras `com.meta.wearable.dat.extra.MWA_INITIATED_DAT_ACTION` = `"beginRegistration"` (+ META_APP_ID / FLOW_ID / PROTOCOL_VERSION / CANCEL_CALLBACK) and yields a `RegistrationRequest` exposing `continueRegistration(Activity)` / `cancelRegistration(): DatResult<Unit, RegistrationRequestError>` (error enum: INVALID_REQUEST, ALREADY_HANDLED, CONFIGURATION_INVALID, REGISTRATION_FAILED, CANCELLATION_FAILED). CHANGELOG (DAT repo): "for accepting or cancelling registration requests initiated by Meta AI app". It is NOT startRegistration's return path, and this probe does not drive it; the `skynetbridge://` browsable filter already declared on the shell's activity is the project's registered callback scheme (the CameraAccess sample declares the same pattern and wires nothing in onNewIntent), and v0.2.6's shell treats an ACTION_VIEW arrival as a return event worth a state re-read — reported, never auto-continued.
- **Meta's own guidance** (AGENTS.md Step 4 + permissions-registration SKILL.md): initialize first, `Wearables.startRegistration(activity)`, collect `registrationState` + `registrationErrorStream`, wait for REGISTERED before creating sessions. Prerequisites: Meta AI app installed, internet, Bluetooth permission, callback URI scheme in the manifest — all already true for iNet.

### What was built

- **NEW facade `probe/RegistrationProbe.kt`** — the only class allowed to drive registration, invoked reflectively by name exactly like the other facades. Three `@JvmStatic` entry points:
  - `start(Activity)`: ensures init with RuntimeInitProbe's exact semantics (success / ALREADY_INITIALIZED ⇒ proceed; typed failure ⇒ thrown as `IllegalStateException` carrying the WearablesError name + description with the SDK cause chained; CrashGuard re-asserted after initialize); arms the collectors (below); snapshots the pre-call state; calls the real **`Wearables.startRegistration(activity)`** raw (throws propagate to the shell's handler untouched); then settle-waits up to 750 ms for the state to move or an error to land. The report states the before/now state, dev mode, device count, whether the state moved to REGISTERING (= the SDK accepted and launched the Meta AI handoff intent — named in the text), and the buffered error/transition record.
  - `onActivityResult(Activity, Int, Int, String)`: the return report. Waits up to 2.5 s while the state is REGISTERING (the ACDC handshake can land a beat after the activity returns), then re-reads state/dev mode/devices and prints an explicit VERDICT line (REGISTERED / still REGISTERING / not registered), alongside the raw requestCode/resultCode/data summary it was handed.
  - `reportState(Activity)`: the callback-scheme return report — a pure re-read + verdict, driving nothing.
  - **Collectors**: a facade-owned `CoroutineScope(SupervisorJob + Dispatchers.Default)` armed exactly once (AtomicBoolean) collects `Wearables.registrationState` into a capped transition log ("HH:mm:ss STATE") and `Wearables.registrationErrorStream` into a last-error slot (name + description + timestamp). Verified present in the compiled output at the bytecode level: `RegistrationProbe$startCollectors$1` invokes `getRegistrationState` + `Flow.collect`; `$startCollectors$2` invokes `Wearables.getRegistrationErrorStream(): Flow` + `Flow.collect`.
- **Shell (`MainActivity.kt`)** — purity preserved (framework `android.*` only; the quarantine was never about framework classes):
  - Fourth button **"Test: start registration"** → `runRegistrationProbe()`: `Class.forName` + reflective `start(Activity)` with `this`; failure handling byte-for-byte the established handler shape (full chain + stack in the log, red one-liner, CrashGuard record). Success sets the state line to "registration STARTED by probe (complete it in the Meta AI app)".
  - **`onActivityResult` override** (framework callback): logs the raw outcome (requestCode, resultCode, and a plain-text data summary — action, data URI, extras with simple values printed and complex values shown as type names), then reflectively forwards to the facade's `onActivityResult` and appends its report as "RETURN REPORT [registration]". The raw outcome is logged even if the facade fails to load, so a return is never invisible.
  - **Callback-intent handling**: `onCreate` (after the shell is built) and `onNewIntent` check for `ACTION_VIEW` arrivals (the manifest's `skynetbridge://` filter) and forward to the facade's `reportState` as "RETURN REPORT [registration callback]". ACTION_MAIN launcher arrivals are ignored.
  - The facade's fully-qualified name lives in ONE string literal (a private property), keeping the class-graph positive control at exactly 1 occurrence. Build line now reads "Build: 0.2.6-v1-registration (versionCode 8)"; the class doc describes the fourth button and the plain-Activity result mechanics.
- **Versions bumped in the three customary places**: `build-manual.sh` aapt2 link (`--version-code 8 --version-name "0.2.6-v1-registration"`), Gradle `defaultConfig` (versionCode 8 / `0.2.6-v1-registration`), and the shell's build line. Nothing else changed: no dependency moves (the resolved graph is v0.2.5's, coroutines 1.11.0 included), no manifest changes (the `<queries>` block, `ACDCRegistrationService` declaration and callback-scheme filter from v0.2.2 already cover this flow), no other facade or source touched.

### Gate evidence (mandatory suite, run against the final signed APK)

- **Dex payload greps** — both dexes extracted from the final APK (`classes.dex` 7,165,444 B, unchanged from v0.2.5; `classes2.dex` **7,206,296 B**, grew from v0.2.5's 7,187,752 by the new facade + shell additions). `classes2.dex` contains ALL of: `Lcom/skynet/raybanbridge/MainActivity;` ✓, `Lcom/skynet/raybanbridge/BridgeApplication;` ✓, `Lcom/skynet/raybanbridge/probe/DatEngineProbe;` ✓, `Lcom/skynet/raybanbridge/probe/MockKitProbe;` ✓, `Lcom/skynet/raybanbridge/probe/RuntimeInitProbe;` ✓, and the new `Lcom/skynet/raybanbridge/probe/RegistrationProbe;` ✓ (classes.dex holds none of them, as before).
- **Class-graph gate** (pre-dex `out/app-classes.jar`, same method as every probe build): `MainActivity.class` — "meta/wearable" **0**, "androidx" **0**, "kotlinx" **0**; `BridgeApplication.class` — 0/0/0. Positive controls in MainActivity.class: `android/app/Activity` (2), and exactly one occurrence each of `probe.DatEngineProbe`, `probe.MockKitProbe`, `probe.RuntimeInitProbe`, and the new `probe.RegistrationProbe` — the only ways the shell knows the facades. The facade carries the DAT references instead: `RegistrationProbe.class` has 12 "meta/wearable" and 15 "kotlinx" occurrences, including `startRegistration` (2), exactly as designed — and its collector lambdas invoke the state/error-stream getters (bytecode-checked above).
- **`apksigner verify`: PASS** — signer cert SHA-256 `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b`, identical to v0.2.1–v0.2.5 (in-place update preserved).
- **`aapt2 dump badging`**: `package: name='com.skynet.raybanbridge' versionCode='8' versionName='0.2.6-v1-registration'`, `application-label:'iNet'`, targetSdkVersion 35.
- **Compiled manifest** (`aapt2 dump xmltree`): meta-data `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓ (CLIENT_TOKEN meta-data, `<queries>`, ACDCRegistrationService and the callback-scheme intent-filter unchanged).
- **Final APK**: `manual-build/app-debug.apk`, **27,938,785 bytes** (the same byte count as v0.2.5 — zipalign padding absorbed the 18,544 B dex growth, the same coincidence documented for v0.2.3→v0.2.4; the hashes prove it is the new artifact), **sha256 `15efd698e65794217f2c0f65ec8bd8bd7e147fadc9b73a2a99553ac6dafdec32`**, md5 `1a6e6522d261bdcf8115a1d80b2e4c4c`. Copy at `~/workspace/your_files/iNet-registration.apk` — sha256 verified identical after copy.
- Build hygiene note: d8 warnings 8,499 vs v0.2.5's 8,487 — same informational classes (the pre-existing "malformed kotlin.Metadata 2.2.0" notices), the delta tracks the slightly larger app graph. Warnings, not errors; the build completed, signed, and verified.

### What Kenn's run will mean (result-decision tree)

Kenn installs v0.2.6 over v0.2.5 and taps the fourth button, **"Test: start registration"**. (If he taps it before ever tapping "Test: initialize runtime" in this process, the facade initializes first with the same semantics — the report still leads with the registration outcome.)

- **PROBE OK + state moves to REGISTERING + the Meta AI app opens** → the handoff intent launched exactly as the SDK designed it. **Kenn's irreducible sliver, pre-staged:** the Meta AI app (Stella) presents the DAT connection flow for "iNet" — he confirms/connects the app there (his Meta account, his glasses, his consent taps; nobody else can make them), possibly including picking the glasses and confirming the link. When the flow finishes he returns to iNet (Back or the flow's own return), and the screen gains a **RETURN REPORT**: raw result + re-read state. What happens next depends on that report:
  - **RETURN REPORT verdict REGISTERED** → registration is complete end-to-end; the whole pre-session stack (class graphs → initialize → registration) is proven on real hardware. **Next build is session creation** (`Wearables.createSession` behind the same guarded pattern, then capabilities). Dev mode in the report may also flip true — the SDK records it from the flow (Kenn's Meta AI Developer Mode is ON; the ComponentActivity-only `is_dev_mode` result-extra handling doesn't run for our plain-Activity shell, so the report's dev-mode line comes from the live `Wearables.isDevMode` read, which is the value that matters).
  - **RETURN REPORT verdict "still REGISTERING"** → the user-side flow finished but the ACDC handshake hadn't landed within the 2.5 s wait. Not a failure: re-check moments later via "Test: initialize runtime" (its report re-reads registration state). If it settles to REGISTERED, treat as the REGISTERED branch with a slow handshake.
  - **RETURN REPORT verdict "not registered" + RESULT_CANCELED** → Kenn (or the Meta AI app) backed out of the flow. Note the SDK's typed cancel handling (`FAILED_TO_REGISTER`) runs ONLY for ComponentActivity hosts, so for this shell a cancel surfaces as exactly this: a canceled result and a state that never left/went back — **by design of the SDK's fallback path, not a defect**. Re-tap the button and complete the flow.
- **PROBE OK + state already REGISTERED at tap time** → the glasses were registered to this app already (e.g. a prior flow completed silently); the report says so and no handoff is needed. Next build is session creation.
- **PROBE OK but the state never moves and an error appears in the report's error line** → the typed error names the target: **META_AI_NOT_INSTALLED** ⇒ the `fb-viewapp://stella/...` intent did not resolve — Meta AI app missing, or too old (the intent demands minMWAVersion 292), or package visibility regressed (the `<queries>` restoration is in the manifest, so this would indict the Meta AI app itself); **ALREADY_REGISTERED** ⇒ same as the already-registered branch; **FAILED_TO_REGISTER / UNKNOWN** ⇒ the SDK refused without a launch — its description text in the report is the lead, and the transition log shows where the state went.
- **PROBE FAILED with a thrown exception** → the stack trace names the failing step: an initialize typed failure is rethrown as `IllegalStateException: Wearables.initialize returned a typed failure before registration could start: <WearablesError> — <description>` (the init layer regressed — follow that error); a throw out of `startRegistration` itself (frames in RegistrationManagerImpl / IntentLauncherImpl) indicts the launch machinery — attestation/config reads live there, and the CLIENT_TOKEN placeholder ("PENDING_FROM_DEV_CENTER"), cleared for init in the v0.2.2 audit, becomes a live suspect for the registration leg specifically, since the intent carries attestation material.
- **PROBE FAILED at facade load (ClassNotFound / NoSuchMethod)** → the packaging regressed, not the flow: the gate evidence in this section proves the shipped artifact carries the facade, so this outcome would mean the installed APK is not this artifact.
- **Shell won't open at all** → unchanged from prior trees: below the activity class graph (the shell's graph is proven identical in kind to v0.2.5 — framework classes plus one added facade-name string and the framework onActivityResult/onNewIntent overrides), and the hunt leaves DAT entirely.


## R-CLASS FIX — v0.2.7-v1-rclass (versionCode 9): the library R classes AGP would have generated, generated

**Built 2026-10-22** (after Kenn's v0.2.6 field run, 08:14 ET: buttons 1–3 fine, button 4 drove the real `Wearables.startRegistration` chain and died inside the SDK — see below). Same manual pipeline (`manual-build/build-manual.sh`, log `build-rclass-run.txt`), same package, label "iNet", icon, and debug cert as v0.2.3–v0.2.6 — installs as an in-place update over v0.2.6. **No app-source logic changed, no dependency version changed, no manifest change.** The only source edits are the three customary version bumps (pipeline link flags, Gradle `defaultConfig`, shell build line) plus the pipeline's new R-generation steps.

### The field failure (v0.2.6, Kenn's screenshot 08:14 ET)

`RegistrationProbe.start` → `Wearables.startRegistration` → `WearablesImpl.startRegistration` → `RegistrationManagerImpl.askRegistration` → `setRegistration` → `RegistrationManagerImpl$WhenMappings.<clinit>` → **`RegistrationError.<clinit>` → `NoClassDefFoundError: Failed resolution of: Lcom/meta/wearable/dat/core/R$string;`** (caused by `ClassNotFoundException: com.meta.wearable.dat.core.R$string`). The SDK reached for its own error-string table to build a `RegistrationError` — and the class did not exist in our APK. The registration flow never launched; this is a packaging defect, not an SDK refusal, not a Meta AI problem.

### Root cause — proven, in three exhibits

1. **The shipped v0.2.6 dexes define ZERO library R classes.** An exact raw-dex audit (type_ids vs class_defs, parser over the dex format — not string greps) of the v0.2.6 APK: **20 `R$` classes referenced, 0 defined, 20 unresolved; 233 `R$` fields referenced, 233 unresolved.** The 20 are the five packaged mwdat modules (`core`, `camera`, `inputs`, `speech`, `mockdevice` — `R$string` each) plus androidx (`core` ×9: attr/color/dimen/drawable/id/integer/layout/string/styleable; `activity` R$id; `core.viewtree` R$id; `lifecycle.runtime` R$id; `lifecycle.viewmodel` R$id; `savedstate` R$id; `startup` R$string). The defect was never just the one field Kenn hit — **every** library R reference in the APK was dangling; `RegistrationError.<clinit>` was simply the first one a probe path executed. (AndroidX's notification/lifecycle code reads its R fields on paths the probes have not yet exercised — they would have failed the same way.)
2. **The AAR carries symbols, not classes.** `mwdat-core-1.0.1.aar` ships `R.txt` (3,919 B, 64 lines — every line a `dat_error_*` string, e.g. `int string dat_error_registration_meta_ai_not_installed 0x7f01000e`, exactly the fields `RegistrationError.<clinit>` reads) and a `classes.jar` of 2,578 classes containing **zero** R classes (count verified). This is by AAR design: the *consuming app's* build generates each library's R class from its R.txt, in the library's own package, with IDs from the app's final linked resource table.
3. **Our pipeline never performed that step — and never linked library resources at all.** The v0.2.6 `aapt2 link` consumed only the app's own `res.zip`: no library `res/` was compiled or linked (the final APK's resource table contains **0** `dat_error_*` entries — verified by `aapt2 dump resources`), and no `--extra-packages`/`--java` generation was requested. Same defect family as v0.2.5's first-seen-wins coroutines pin: a step a real build performs silently, missing from the hand-rolled pipeline.

### The fix — the standard AGP/aapt2 route, inside the existing pipeline

`build-manual.sh` now performs the missing generation step the way AGP does:

- **Step 1 (extract):** for every AAR extracted from the resolved graph, record (a) its manifest package if it ships an `R.txt` (→ the R-generation list) and (b) its `res/` dir if non-empty (→ the link list). Result this build: **32 AARs carry an R.txt → 32 unique library packages; 26 AARs ship resources.**
- **Step 2 (link):** every library `res/` is compiled with `aapt2 compile` and linked together with the app's resources (app first, `--auto-add-overlay`), and the link runs with **`--extra-packages <all 32 library packages>` + `--java out/gen` + `--output-text-symbols out/final-symbols.txt`** — aapt2 emits an `R.java` for the app package and each library package, with values taken from the **final linked table**.
- **Step 3b (new):** the 33 generated `R.java` files (32 library + app) are compiled with the toolchain JDK's `javac` into `r-classes.jar`, which joins the d8 inputs in step 4 — so the packaged dexes **define** every generated R class.

**Value consistency is by construction, and it is proven, not asserted.** For the exact field that killed v0.2.6: the AAR's R.txt placeholder is `0x7f01000e`; aapt2's final linked table (`out/final-symbols.txt`) assigns `dat_error_registration_meta_ai_not_installed = 0x7f090037`; the generated `R.java` field reads `0x7f090037`; and `aapt2 dump resources` on the shipped APK shows `resource 0x7f090037 string/dat_error_registration_meta_ai_not_installed`. Generated value = packaged table value; the R.txt placeholder was used only as the symbol *list*, never as values.

Scope notes, stated plainly:
- **Only 5 of the 7 mwdat modules are in this app's resolved graph** (core, camera, inputs, speech, mockdevice — as in every build since v1). `mwdat-display` and `mwdat-motion` AARs exist in the toolchain store but are not dependencies of this app: their classes are not packaged, and the audit confirms **zero** references to their R classes. There is nothing to generate for them; if a future build adds them as dependencies, step 1 picks them up automatically (the generation list is derived from the extracted graph, not hard-coded).
- aapt2's `--extra-packages` generation is *unscoped*: each library's R class carries the full final-table symbol set for its resource types (e.g. dat-core's generated `R.java` holds 342 constants, including the other modules' `dat_error_*` strings). This matches legacy aapt behavior, is harmless (extra int constants), and guarantees every R.txt symbol a library was compiled against is present with the correct value.
- AAR **assets** remain unpackaged (two AARs ship `assets/` dirs: mwdat-mockdevice, soloader) — pre-existing behavior, unchanged by this fix and outside its scope; noted here so the gap is on the record.

### Gate evidence (mandatory suite, run against the final signed APK)

- **Gate 1 — DEFINITION, dexdump level.** `dexdump` on the shipped `classes2.dex` shows the class-definition block:
  `Class descriptor : 'Lcom/meta/wearable/dat/core/R$string;'` — `PUBLIC FINAL`, superclass `Ljava/lang/Object;`, static fields carrying values (field #0 `androidx_startup = 2131296256`, …). The same descriptor-level definitions are present for the other four mwdat `R$string` classes (camera, core, inputs, mockdevice, speech) and for every other generated R class (audit below).
- **Gate 2 — REFERENCE→DEFINITION AUDIT (the defect-class closer).** Exact raw-dex audit of the final APK (both dexes; a type/field counts as *referenced* if its id is interned in the dex, *defined* if a class_def / class_data field list carries it):
  | | v0.2.6 (before) | v0.2.7 (after) |
  |---|---|---|
  | `R$` classes referenced | 20 | 363 |
  | `R$` classes defined | 0 | 363 |
  | `R$` classes unresolved | **20** | **0** |
  | `R$` fields referenced | 233 | 10,923 |
  | `R$` fields unresolved | **233** | **0** |
  (The "referenced" totals grow because the generated R classes are now themselves in the dex type pool. The load-bearing numbers are the unresolved rows — and a dedicated cross-check confirms **all 233 fields v0.2.6's bytecode actually referenced are defined in v0.2.7, 0 missing**.) The 33 outer `R` classes (32 library packages + the app's own) are likewise all defined. Unresolved = 0 at both class and field level; nothing left to name.
- **Gate 3 — RESOURCE RESOLUTION: the `dat_error_*` strings RESOLVE in the final APK.** Because the fix links library resources (not just generates classes), the strings are real table entries now: `aapt2 dump resources` finds **114 `dat_error_*` resources** in the shipped APK (v0.2.6: **0**), including `resource 0x7f090037 string/dat_error_registration_meta_ai_not_installed`, whose packaged value is string-pool #134: *"The Meta AI app is not installed on the device, which is required for registration"* — the AAR's own text. When `RegistrationError` formats an error on-device, `Resources.getString(id)` will find its text.
- **Gate 4 — shell integrity.** Class-graph gate (pre-dex `out/app-classes.jar`, same method as v0.2.3–v0.2.6): `MainActivity.class` — "meta/wearable" **0**, "androidx" **0**, "kotlinx" **0**; `BridgeApplication.class` — 0/0/0. Positive controls: exactly one occurrence each of `probe.DatEngineProbe`, `probe.MockKitProbe`, `probe.RuntimeInitProbe`, `probe.RegistrationProbe` in MainActivity.class; `RegistrationProbe.class` still carries 12 "meta/wearable" / 15 "kotlinx" occurrences — byte-for-byte the v0.2.6 facade profile (facade and shell logic untouched; buttons 1–3 behavior unchanged, button 4 returns its VERDICT lines as before). All **6 app descriptors** present in the final dexes (MainActivity, BridgeApplication, DatEngineProbe, MockKitProbe, RuntimeInitProbe, RegistrationProbe).
- **Gate 5 — signature / badging / artifact.** `apksigner verify`: **PASS** — signer cert SHA-256 `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b`, identical to v0.2.1–v0.2.6 (in-place update preserved). `aapt2 dump badging`: `package: name='com.skynet.raybanbridge' versionCode='9' versionName='0.2.7-v1-rclass'`, `application-label:'iNet'`, targetSdkVersion 35. Compiled manifest: meta-data `com.meta.wearable.mwdat.APPLICATION_ID` = `"2468383600319627"` ✓ (unchanged). Dexes: `classes.dex` 7,317,252 B, `classes2.dex` 7,246,792 B (v0.2.6: 7,165,444 / 7,206,296 — grew by the generated R classes and d8's re-split). d8 warnings 8,499 — identical count to v0.2.6, same informational classes.
- **Final APK**: `manual-build/app-debug.apk`, **28,625,392 bytes** (v0.2.6: 27,938,785 — the +686,607 B delta is the newly linked library resources + generated R classes), **sha256 `eb2c0397b145b87b7e0472907137916e45d9818fa5a1bd997df183df88cb89b6`**. Copy at `~/workspace/your_files/iNet-rclass.apk` — sha256 verified identical after copy, and the copy itself passes `apksigner verify`.

### What Kenn's re-run will mean (result-decision tree)

Kenn installs v0.2.7 over v0.2.6 and taps the fourth button, **"Test: start registration"**. The R-class wall is proven gone at the artifact level, so the flow now proceeds to whatever is genuinely next:

- **State moves to REGISTERING + the Meta AI app opens** → the handoff intent launched. Kenn completes the connect/consent flow inside Meta AI (his sliver, as pre-staged in the v0.2.6 tree) and returns; the RETURN REPORT's verdict governs: **REGISTERED** → the whole pre-session stack is proven on real hardware; next build is session creation. **Still REGISTERING** → slow ACDC handshake; re-read via button 3. **Not registered + RESULT_CANCELED** → he backed out in Meta AI (the SDK's typed cancel handling is ComponentActivity-only — by SDK design, not a defect).
- **An error lands in the report instead** → it is now a *real* SDK verdict with working error formatting behind it (the class whose absence killed v0.2.6 is the one that formats these): **META_AI_NOT_INSTALLED** ⇒ the `fb-viewapp://stella/...` intent didn't resolve (Meta AI app missing/too old — minMWAVersion 292 — or visibility regressed); **ALREADY_REGISTERED** ⇒ treat as registered; **FAILED_TO_REGISTER / UNKNOWN** ⇒ follow the description text, now guaranteed readable end-to-end.
- **PROBE FAILED with a thrown exception whose frames sit in RegistrationManagerImpl / IntentLauncherImpl** → the launch machinery itself threw (attestation/config reads live there; the CLIENT_TOKEN placeholder, cleared for init in the v0.2.2 audit, is the named suspect for the registration leg specifically, since the intent carries attestation material).
- **Any NEW NoClassDefFoundError / NoSuchFieldError naming an `R$` class** → would contradict this section's audit (0 unresolved at class and field level across both dexes); if it appears, the installed APK is not this artifact — verify the sha256 of the installed build first.
- **Shell won't open at all** → unchanged from prior trees: below the activity class graph (proven identical in kind to v0.2.6 — same sources, version strings aside), and the hunt leaves DAT entirely.

---

## 2026-10-24 — Postmortem: the isDevMode wall was (partly) ours

> "Two days, three phones, a portal, a red team — and the culprit was sitting in our own manifest wearing a name tag that said PENDING."

Filed publicly because every trap below is one any DAT developer can fall into. All findings verified against our own code, the shipped binaries (aapt), and device logs.

**Where we ended up (2026-10-24, Seeker host, Meta AI v296):** Developer Mode ON, iNet REGISTERED via Meta AI's real consent surface ("Connect Skynet Bridge to your Meta devices? — Release channel Beta 1.0.0"), ACDC logcat: "secure registration successful", known devices: 1 — and `Wearables.isDevMode` still reads **false**, with the devicesMetadata feed frozen on a single static DISCONNECTED record. Registration is solved; two gates remain (the flag, the feed).

**What the audit found in our own house (ZERO-BUILD-AUDIT, 2026-10-24):**

1. **No shipped build was ever a true 0/0 build.** v0.4.1 ("devmode-id0") = APPLICATION_ID `0` + CLIENT_TOKEN still the literal placeholder `PENDING_FROM_DEV_CENTER`. v0.4.2/v0.4.3 = real APPLICATION_ID + real token. Meta's documented DevMode precondition (ID and token both 0) had never once been satisfied.
2. **The registration host was the wrong activity type.** Our MainActivity extends plain `android.app.Activity`. With a plain-Activity host, the SDK launches the registration handoff via `startActivityForResult` and consumes no result itself — so the activity-result path that sets `isDevMode` never executes. Meta's CameraAccess sample passes a `ComponentActivity`. Our own code comments described this behavior; nobody connected it to the flag.
3. **`startRegistration()` was called unconditionally.** Per facebook/meta-wearables-dat-android#192, calling it while already REGISTERED can silently reset `isDevMode`. Our "Already REGISTERED" handling was report text, not a guard — the call fired anyway.
4. **`isDevMode` was never gated on.** It was read into every probe panel for display; session creation never checked it.

**Outside references that named the mechanism (Meta staff, via the linked issues):** #197 — `isDevMode` is set only from the result Meta AI returns to the activity passed to `startRegistration()`; registration state and the dev flag are separate steps, and REGISTERED + isDevMode=false is not an expected state on a Developer Mode build; while false, sessions reach the glasses marked non-DevMode. #214 — registration failures on specific Meta AI builds can also be Meta-side; record your Meta AI build number with every result.

**One more tell worth checking in your own runs:** our successful consent named a *release channel*. Developer Mode apps skip attestation and release channels entirely; the release-channel connect flow (tester invite, channel pick) is the Developer-Mode-OFF path for a real App ID. If your consent surface names a channel, you may have registered as a production app — in which case isDevMode=false is an accurate reading, not a lost flag.

**The fix in flight (v0.4.4-zero):** APPLICATION_ID `"0"` and CLIENT_TOKEN `"0"` (both via string resources — a bare `"0"` literal hits an integer-typing trap), MainActivity migrated to `ComponentActivity`, a REGISTERED guard in front of `startRegistration()`, an `isDevMode` gate in front of session creation, and a clean-state run protocol (clear storage, re-grant, one registration call, then read the flag warm).

**Traps for other DAT developers, collected the expensive way:**

- Audit the values in the **shipped binary** (aapt), not your source comments. Our manifest comment still claimed the token was a placeholder weeks after it wasn't — and vice versa in the build that mattered.
- In Meta AI v296, the Developer Mode surface is **hidden behind tapping the App version row** in Settings → App Info (the Android build-number ritual). Two exhaustive automated UI sweeps read that page and concluded the surface did not exist. "The row isn't there" is not "the surface doesn't exist."
- A DAT app's **Nearby devices** grant must be full **Allow**; "Allow only while using the app" silently starves the metadata feed (registration still works, which makes it a nasty decoy).
- Meta AI supports **one Developer Mode app at a time** — registering a second DAT app (including the official sample) can reset the first app's dev registration. Sequence your control tests accordingly.
