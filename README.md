# walking-loop (Napoleon)

`feat-walking-session-loop-v1`, increment 1 — the workstation-routed voice Q&A loop.
Arc: `arch-orchestrator-voice-first-surface-phase-2` (FPSR). Decision record:
`docs/design/walking-session-stt-and-routing-v1.md`.

## What it is

A thin token-gated FastAPI service that runs **natively on Napoleon** and orchestrates
the two primitives already on localhost there — it does not re-implement either:

- STT — `POST http://127.0.0.1:8771/transcribe` (stt-napoleon, Bearer + multipart `file=`)
- inference — `POST http://127.0.0.1:11434/api/chat` (native Ollama, `qwen3.5:9b`)

Both calls are localhost on Napoleon, so the hot loop never crosses VMnet8 and never
flaps. The phone reaches this service directly over the tailnet. The dreambox-1 guest
is not in the path. This is also the answer to `arch-pocket-voice-inference-location-tier3-gated`:
workstation-path inference = Napoleon-local = Tier-3-clean and flap-free.

## Endpoints

- `GET /health` — unauth; reports STT/Ollama reachability + open sessions.
- `POST /session/start` — `{type: "general"|"therapy"}` → `{session_id}`; warms the 9b.
- `POST /turn` — multipart `session_id` + `file` (audio) → `{transcript, reply_text}`.
- `POST /session/end` — `{session_id}` → `{transcript}`; drops the in-memory session.
- `GET /` — the hands-free phone page (served same-origin).

## Tier-3

Napoleon is trusted on-prem; STT + inference never touch a third party. A `therapy`
session here is Tier-3-clean **on a trusted network** (the only therapy mode v1
supports — see the routing matrix). Therapy transcripts stay in memory and are never
written to disk; general sessions may be appended to `~/.walking-loop/sessions/`.
What a transcript is *routed to* afterward is the downstream P3 item
(`arch-walking-session-content-routing`), out of scope here.

## Layout

- code: `C:\Users\Tony\walking-loop\` (server.py, launcher.ps1, install.ps1, requirements.txt) — scp'd from the dreambox repo
- runtime: `C:\Users\Tony\.walking-loop\` (venv, token.txt, sessions/)
- reads the STT bearer from `C:\Users\Tony\.stt\token.txt` (already present from stt-napoleon)

## Bring-up

Repo-first (the repo on dreambox is canonical; Napoleon is not a clone):

1. Land these files into the repo on dreambox-1 at `services/walking-loop-napoleon/`, commit (stage only these paths, never `-A`).
2. scp the four runtime files (server.py, launcher.ps1, install.ps1, requirements.txt) from dreambox → `C:\Users\Tony\walking-loop\` on Napoleon.
3. On Napoleon: `& ~\walking-loop\install.ps1` (venv, deps, local token, firewall, task).
4. `Start-ScheduledTask -TaskName walking-loop` (or `& ~\walking-loop\launcher.ps1` to watch logs).

The `.ps1` files arrive via scp (not a browser download), so they carry no
Zone.Identifier — `Unblock-File` is not needed.

## Token

`install.ps1` generates the loop bearer into `~\.walking-loop\token.txt` (32 hex) and
never prints it. The phone page prompts for it once (stored in the phone's
localStorage). Read it on Napoleon and enter it into the page — don't paste it into a
chat. The tailnet-scoped firewall (100.64.0.0/10) is the real boundary; the token is
defense-in-depth.

This **supersedes** the s136 handoff's "remaining bit (b): scp the STT token to
dreambox-1." With the loop on Napoleon, it reads the STT token locally — no scp, no
token in the hot path.

## Green bar

- `Invoke-RestMethod http://localhost:8772/health` on Napoleon → `ok`, `stt_health: true`, `ollama_up: true`.
- From dreambox-1: `curl -s http://napoleon:8772/health` → reachable over the tailnet (no VMnet8).
- Bad token → 401.
- From the phone on the trusted network: open `http://napoleon:8772/`, enter the token, start a session, do one real turn → confirms the loop end-to-end **and** doubles as the stt-primitive's outstanding real-words accuracy check (vs the s136 tone smoke-test).

## Honest v1 limits

- Browser hands-free is screen-on only — backgrounding / screen-sleep drops the session (`dreambox-2026-034`). True background hands-free likely needs a native shell; out of v1 scope.
- The on-page silence detection (`SILENCE_MS`/`RMS_GATE` in server.py's embedded page) is crude and will need tuning against real walking noise (wind/traffic).
- 9b warm during a walk contends with the nomic embedder on the 4080; `keep_alive=5m` bounds it to the walk, and an indexer run mid-walk may queue/retry. Acceptable.
- The fully-on-phone local loop (on-device STT + inference + TTS) is increment 2, where pocket-voice decision (a) — the exact on-device model — finalizes. This increment is the reliable workstation fallback and the Tier-3 therapy path.
