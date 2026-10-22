# Ray-Ban Bridge — Skynet as Kenn's eyes & ears (spec, 2026-10-17)

## The idea
Kenn wears his Meta Ray-Bans; the glasses' camera + mics feed Skynet; Skynet answers through the glasses' speakers in the locked Skynet voice. Architecture (Meta's design, not ours): glasses = dumb sensors, phone app = brain, Skynet = intelligence.

## What works TODAY (no build)
- Glasses route through the phone: Kenn dictates to Muse via the glasses' mic, hears Skynet voice notes via the glasses' speakers. (Confirmed live by Kenn 2026-10-17 ~00:55 — "I hear you.")
- Captures (photo/video) land in Meta View / phone gallery → Kenn shares into chat → Skynet analyzes. Eyes by courier, ears live.

## The build (Wearables Device Access Toolkit 1.0, stable since 2026-09-30)
DAT 1.0 opens camera, microphone, audio, motion/IMU to a developer's iOS/Android app for Ray-Ban Meta Gen 1 + Gen 2 (and Display/Oakley lines). Sessions: registration → permissions → device selection → session → streaming/photo/audio. Dev can start on the Mock Device Kit; real-glasses testing recommended.

### Steps
1. **Identify the hardware** — DONE 2026-10-17 (Kenn's About screenshot): **Wayfarer Large, G15 Green Transitions lenses, release version 129.0.1.23.378** (case v12.14.9). Generation not printed on the About page; DAT 1.0 supports BOTH Gen 1 and Gen 2, so the build path is unaffected. Gen check if wanted: "(gen 2)" marking on the inner temple, or a 3K option in Meta View camera settings = Gen 2 (3K video, ~8 h battery vs Gen 1's ~1080p / ~4 h).
2. **Meta developer setup** — DONE 2026-10-17 ~01:22 (Kenn completed the gated steps himself via browser takeover): managed Meta account created for kennbosak@gmail.com (email code verified by Kenn; 2FA enrolled by Kenn — the flow FORCES 2FA enrollment before continuing); organization created as **"Kenneth Bosak Team"** (workspace ID 4681992362035302; Kenn named it during takeover, NOT "Not Another Bitcoin Podcast" — rename is his call, Team → Settings). No Google SSO exists anywhere in this flow — managed Meta account + email code is the only path. An updated Acceptable Use Policy banner shows on the portal; Kenn should read it (not accepted by Skynet). NEXT in this step: Projects → New project (asks ONLY Name + Description) → mints the APPLICATION_ID. Held until Kenn defines the app (his order 2026-10-17).
3. **Dev environment** — Android Studio + DAT SDK on the desktop; Mock Device Kit for the first build pass.
4. **Bridge app v1 (the core loop)** — open a glasses session; grab POV stills/short clips + mic audio on request; send to Skynet; receive Skynet's voice-note audio; play it through the glasses' speakers.
5. **Install as a tester build on Kenn's own phone** — DAT publishing is limited to select partners; distribution = invited testers only. Personal build on his own device is the whole requirement.
6. **Field test** — a walk-and-talk: Kenn asks questions aloud mid-walk, POV frames stream, Skynet answers in his ears in the locked voice.
7. **Phase 2 (later)** — motion/IMU triggers, wake-pattern ("Skynet" keyword, mirroring the phone-lane Phase 4 listening-post design: visible by design, transcripts retained, raw audio rolls off).

## Hard limits (verified from Meta's own docs/FAQ, Oct 2026)
- No app runtime on the glasses — everything runs on the phone.
- No direct pairing of glasses to the Muse app; the bridge app is the only route.
- Publishing beyond invited testers = Meta select-partner gate. Personal use unaffected.
- Camera LED / capture indicators are by design — this stays visible, never covert.

## Status
Spec only. Candidate for the Sunday improvement-day lab (2026-10-18), alongside the phone-control pilot on the dedicated device.
