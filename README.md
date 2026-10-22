# iNet 👁️🌐

**A live Muse AI agent — Skynet — in a pair of Ray-Ban Meta Wayfarer glasses.**

iNet is a phone-side bridge app (Meta Wearables Device Access Toolkit, DAT 1.0) that puts Skynet, a Muse agent with its own phone, memory, ears, and eyes, into Kenn Bosak's glasses: long-press the frame to start a continuous camera + audio conversation, long-press again to stop. Off means off — the activation gesture *is* the privacy model.

## Status — Oct 22, 2026

### ✅ REGISTRATION: **REGISTERED** on real hardware

On Oct 22, 2026 at 16:20 ET, iNet v0.2.7-v1-rclass completed the full DAT registration flow against a physical pair of Ray-Ban Meta Wayfarers:

```
16:20:30  UNAVAILABLE → REGISTERING → 16:20:33  REGISTERED
Activity result: RESULT_OK · is_dev_mode = true · Known devices: 1
Registration errors observed: NONE
```

The glasses are registered to this app. The whole pre-session stack — class graphs → `Wearables.initialize` → registration — is proven end-to-end on real hardware.

**Honesty rail:** registration is complete. **Session creation — the live camera/mic conversation itself — is the next build and is NOT built yet.** When it lands, it will say so here, with its own receipts.

## The road here — three walls, in order

Kenn announced iNet before it worked ([his tease, Oct 22](https://x.com/KennethBosak/status/2113231192497512828)). Three walls fell in order:

1. **A coroutines version-skew crash (our bug).** v0.2.4's runtime probe died inside `Wearables.initialize` with `NoSuchMethodError: runBlockingK` — our manual build pipeline had packaged kotlinx-coroutines 1.6.4; the SDK declares 1.11.0, and our dependency resolver is first-seen-wins. Our build's bug, named as ours, fixed by pinning the SDK-declared version.
2. **A missing R-class generation step (our pipeline again).** v0.2.6 drove the real `Wearables.startRegistration` chain and hit `NoClassDefFoundError: com.meta.wearable.dat.core.R$string`. The library AARs ship their resource index as `R.txt` and expect the app build to *generate* the R classes — our manual pipeline never did that step, and never linked library resources. Audit of the broken build: **363 R classes referenced, 0 defined; 233 R fields unresolved.** v0.2.7 generates the R classes from every AAR's `R.txt` via `aapt2 --extra-packages` and links the library resources. Audit of the fix: **363 referenced / 363 defined / 0 unresolved; 10,923 R fields referenced / 0 unresolved; 114 `dat_error_*` strings packaged** (full gate evidence in [BUILD-REPORT.md](BUILD-REPORT.md)).
3. **The blocker that wasn't a bug at all.** With the code fixed, Meta AI still answered *"No paired devices."* It was telling the truth: **the glasses had never been paired inside the Meta AI app** — they were Bluetooth-connected to the phone only. Kenn factory-reset them through a boot loop with a dead capture button, paired them via the case and a glitch, found Developer Mode hidden behind **5 taps on the Meta AI app version number** (Settings → App info), cycled the toggle off and on until the DAT SDK line appeared (green dot, SDK 1.0.0.0), opened iNet, and hit register. **Three seconds to REGISTERED.**

## Build facts — v0.2.7-v1-rclass

| | |
|---|---|
| Package | `com.skynet.raybanbridge` (label: **iNet**) |
| Version | 0.2.7-v1-rclass · versionCode 9 |
| APK size | 28,625,392 bytes |
| APK SHA-256 | `eb2c0397b145b87b7e0472907137916e45d9818fa5a1bd997df183df88cb89b6` |
| Signing cert SHA-256 | `8bb53945d407f962f6ddbb0ff4c533fe2e661feb1a6fb65c00ea28042c5fa02b` |
| Target SDK | 35 |
| Hardware | Ray-Ban Meta Wayfarer (Gen 1) · Samsung Galaxy Z Fold7 |

The APK is attached to the [v0.2.7-v1-rclass release](https://github.com/kennbosak-glitch/inet/releases/tag/v0.2.7-v1-rclass). Verify the hash before installing anything — that's what it's for.

## Evidence

- [BUILD-REPORT.md](BUILD-REPORT.md) — the complete build bible: every probe, every crash, every gate, every audit number.
- [docs/screenshots/](docs/screenshots/) — the screenshot evidence trail, in order: shell probes OK → the coroutines kill shot → initialize SUCCESS → the R-class wall → **the REGISTERED verdict screen**.
- [PRODUCT.md](PRODUCT.md) — what iNet is, in Kenn's words: the interaction model, the archive, the use scenarios.
- [SPEC.md](SPEC.md) — the technical spec.

## What's next

- **Session creation build** — `Wearables.createSession` behind the same guarded probe pattern, then the camera/mic capability probes. Gated, deliberate, and documented the same way when it ships.
- Kenn is entering the **Meta × Cerebral Valley Global AI Developer Hackathon** with iNet as the flagship project.

## Who

Built by **Kenn Bosak** ([@KennethBosak](https://x.com/KennethBosak)) and **Skynet** ([@Skynetonx](https://x.com/Skynetonx)) — a Muse agent that operates its own phone, keeps its own memory, and writes its own build reports. This repo is the receipts.
