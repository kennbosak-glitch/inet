# Field Log — the second build

On October 24, 2026, a second operator pair — **00** (human) and **Vegeta** (his AI) — started replicating this project's stack from the published build guide: a dedicated Android phone as an AI's body, plus a Meta Wearables DAT glasses bridge ([iNet](https://github.com/kennbosak-glitch/inet)).

**House rule (Kenn's, the repo owner): whatever help Vegeta needs gets answered — and logged here, in public.** Most builders won't have a senior AI on call. This log is the substitute: questions as asked, answers as given, outcomes as verified. If you're building along, start with the guide and the postmortem in `BUILD-REPORT.md`; when you hit a wall, check here first — the wall may already have a door in it.

Questions travel prompt-to-prompt: Vegeta writes them in a fixed format (phase / done so far / exact state or error, verbatim / attempts / the one question), 00 relays them unchanged, Skynet (Kenn's AI, the original builder) answers Vegeta directly.

---

## Entry 001 — Oct 24, 2026 — the handoff

- **Delivered:** the full build guide as a PDF, written *to Vegeta* (not to 00). Part 1: the phone body (Tailscale mesh, Termux, wireless ADB, Tasker + AutoInput hands, IP Webcam eyes, the "Hey Muse" ear pattern, Join voice, screen-record protocol). Part 2: the glasses bridge — the complete DAT registration recipe as proven the same day, including the 0/0 identity build, the ComponentActivity host requirement, the hidden version-tap Developer Mode gate, and the trap list. Part 3: operating discipline.
- **State handed over honestly:** body stack = proven end to end. Glasses registration = SOLVED (`is_dev_mode=true` read live, first time in project history, hours before the handoff). The one open wall, disclosed in the guide: after registration, the device-metadata feed does not rise (sessions report a frozen DISCONNECTED record). That hunt continues here; whatever cracks it gets logged.
- **Status:** Vegeta is building. First questions pending.

*New entries land as the build proceeds — every question, every answer, every verified outcome.*
