# CLAUDE.md

This repo is a fork of MuddyWinds/Readback. We are turning it into a local radio signals pipeline for South Africa.
Full plan: `BUILD_PLAN.md`. Read it once before starting.

## First task: quick demo (target: under one hour)

Goal: show that a live SABC radio stream becomes a list of local events with places.
Do not refactor. Do not strip aviation code yet. Change only what the demo needs.

1. Resolve the direct audio stream URL for Phalaphala FM (SABC, Tshivenda). SABC stations stream online. Confirm with `ffprobe` for 30 seconds. Add one more SABC station as a second feed if quick.
2. Record 10 minutes with ffmpeg to `samples/phalaphala_<timestamp>.mp3`.
3. Transcribe the sample with faster-whisper `medium`, language auto-detect. Save to `samples/<name>.txt` with the detected language.
4. Add a `radio` analysis mode in `backend/core/batcher.py`: a new prompt that returns JSON events with `type, place, time_mentioned, summary, confidence, station`. Event types: outage_water, outage_power, weather, road, protest_strike, crime_safety, health, opportunity, community, other. Keep the ATC prompt untouched.
5. Run the sample transcript through the radio prompt. Save output to `samples/<name>.events.json`.
6. Start the stack with `docker compose up` and point one monitor at the Phalaphala stream for 15 minutes. Confirm transcripts and events reach Postgres.
7. Write `DEMO.md`: stream URLs used, detected language, transcript excerpt (short), events produced, what failed, next step.

## Rules

- Work on a feature branch and open a PR to `main`. No loose branches.
- Never put Claude session links in commits or PR descriptions.
- Never write SAWS weather warnings in your own words. Quote them as received.
- If faster-whisper output is unusable for Tshivenda, record that in `DEMO.md` and stop at step 5. The fallback (Meta MMS Tshivenda adapter) is a separate task.
- If port 5432, 3000 or 8000 is taken on the host, remap the container port in a local override file. Do not stop other services.
- LLM key: use `GEMINI_API_KEY` from `.env` for the demo. Moving to the LiteLLM gateway is a later step.

## Do not read

`frontend/node_modules`, lockfiles, `tests/fixtures`, recorded audio files.
