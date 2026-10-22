# iNet — screenshot evidence trail

The probe hunt, in order. Each shot is from Kenn's Fold7, taken live during the session named. These are the receipts for the GitHub / Reddit / X documentation cascade when the bridge story publishes.

| # | File | When (ET) | What it proves |
|---|------|-----------|----------------|
| 1 | `01-2026-10-22-shell-probes-ok.jpg` | Oct 22, 07:20 | v0.2.3 shell probes: DAT core classes + mock kit load and initialize clean — **PROBE OK ×2**. Class loading exonerated for the v0.2.2 launch crash. |
| 2 | `02-2026-10-22-runtime-init-fails-runblockingk.jpg` | Oct 22, 07:30 | v0.2.4 runtime-init probe: **the kill shot** — `NoSuchMethodError: runBlockingK` in `kotlinx/coroutines/BuildersKt`, thrown inside Meta's ACDC store during `Wearables.initialize`. Root cause: our manual pipeline packaged coroutines 1.6.4; the SDK declares 1.11.0. |
| 3 | `03-2026-10-22-initialize-success.jpg` | Oct 22, 07:49 | v0.2.5 coroutines-fix: **`Wearables.initialize` returned SUCCESS** — full init chain completed. Registration state: UNAVAILABLE, dev mode false, 0 known devices → next build is the registration flow (v0.2.6). |
| 4 | `04-2026-10-22-registration-rclass-missing.jpg` | Oct 22, 08:14 | v0.2.6 registration probe: the button drove the REAL `Wearables.startRegistration` chain and the probe caught the next layer — `NoClassDefFoundError: com.meta.wearable.dat.core.R$string`, thrown from `RegistrationError.<clinit>` inside `RegistrationManagerImpl`. Root cause (Skynet-verified on the artifact): the AAR ships its resource index as `R.txt` and expects the app build to GENERATE the `R` classes — our manual pipeline never did. Zero R definitions in the v0.2.6 dexes, one dangling reference. Fix = v0.2.7 (generate R classes for every AAR in the graph + a reference→definition audit). |
| 5 | `05-registration-registered-2026-10-22.jpg` | Oct 22, 16:20 | **THE VERDICT.** v0.2.7-v1-rclass registration run: state transitions `16:20:30 UNAVAILABLE -> REGISTERING -> 16:20:33 REGISTERED`; activity result RESULT_OK with `is_dev_mode=true`; **Known devices: 1**; no registration error on the error stream. The glasses are registered to iNet on real hardware. The blocker behind the morning's "No paired devices" was real: the glasses had never been paired inside the Meta AI app (Bluetooth-only) — fixed by Kenn the same afternoon (factory reset, in-app pairing, Developer Mode enabled). |

Originals of 01–04 live in `~/workspace/user/files/` (36618_282, 36626_283, 36628_284, 36641_290). Full technical record: `../../app/BUILD-REPORT.md`.

Still to come in this folder as the build lands them: the first live session (camera/mic), and the first frame from the glasses themselves.
