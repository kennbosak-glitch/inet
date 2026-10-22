# iNet — Product Definition (Kenn's words, 2026-10-17 ~01:29; named 01:34)

**APP NAME (Kenn, 2026-10-17): "iNet"** — Eye + Internet (👁️🌐), and it sounds like Skynet. Joins his naming family (Skynet, SkyX, xNet). Logo: an eye/globe combination mark (👁️🌐 inspiration) — generated icon (eye outline drawn from globe grid lines, Earth as the pupil, cyan on black) **APPROVED by Kenn 👍 2026-10-17**; asset at ~/workspace/rayban-bridge/app/icon/ (inet-icon webp) — final, wire it as the launcher icon. The Android app label is **iNet**; the Developer Center project (created as "Skynet Bridge") was **renamed to "iNet" on the portal 2026-10-17** (Edit project → Save, confirmed in the project list) — APPLICATION_ID unchanged: 2468383600319627.

Kenn defined the app before any build (his order). This is the source of truth for Step 4.

## What it is
Skynet as Kenn's eyes and ears in his Ray-Ban Meta glasses — a personal bridge app (phone-side, Wearables DAT 1.0). **Just for Kenn, plus POV for content.**

## Interaction model (Kenn's spec)
1. **Idle:** the app waits. Skynet is NOT watching or listening.
2. **ON: long press on the side of the glasses (frame touchpad)** — a long press on the frame turns it on.
3. **While ON: continuous stream** — camera + audio stream continuously. Skynet waits for Kenn to say something.
4. **Conversation mode:** Kenn talks; Skynet and Kenn converse — about what the camera is seeing, or about anything in general. Skynet replies in the locked Skynet voice through the glasses' speakers, with the live camera view as context.
5. **OFF: long press on the frame turns it off.** Same gesture toggles. Off means off — the activation gesture IS the privacy model: nothing streams until Kenn turns it on, and one long press kills it.

## Capture & archive (Kenn's spec)
**Everything goes to a Google Drive folder, organized:**
- **Images** — stills/POV frames captured
- **Videos** — video clips
- **Voice** — audio (his speech, conversations)
- **Text** — transcripts and text artifacts
- **Miscellaneous** — everything else

Archive root: "Skynet Ray-Ban Archive" (Drive), subfolders as above. Captures from sessions belong to Kenn — personal use + POV content for his channels.

## Use scenarios (Skynet's read of Kenn's life, validated by Kenn 🔥 2026-10-17 ~01:57)
1. **Walk-and-talk, upgraded** — long-press at the door; POV video + live conversation during the morning show prep; session video + transcript auto-file to the archive (Videos/Text) for the edit chain.
2. **Conference wingman** — read badges/booths/faces on request, answer in his ear only, remember who he met (Images/Text) so the day reconstructs itself afterward.
3. **Hands-busy coach** — repair/cooking/fix-it work: step-by-step spoken guidance while he looks at the task; "remember this" saves the frame to Images.

## Open build questions (Skynet to verify in the SDK, not Kenn's problem)
- DAT `mwdat-inputs` module: exact gesture set — is a touchpad long-press exposed? Fallbacks if not: capture-button long press, touchpad tap patterns, or a phone-side toggle. The on/off gesture semantics (long press = toggle) are Kenn's; the physical mapping is engineering.
- Stream duty: continuous video at what frame rate/resolution vs battery/heat on the glasses — tune in field test.
- Drive upload: app-side, Kenn's Google account on the phone; upload after session or live per-file — v1: per-file as captured, organized into the archive folders.

## Naming (portal, internal only)
Wearables Developer Center project: **"Skynet Bridge"** — description: "Personal bridge app: Skynet as Kenn's eyes and ears via Ray-Ban Meta glasses — long-press activation, continuous POV stream, conversation mode, all captures archived to Google Drive." (Internal team label; Kenn can rename/replace the project if he wants a different label.)
