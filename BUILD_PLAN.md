# BUILD_PLAN: Local Radio Signals (Vhembe proof)

Goal: prove that local radio can seed a local information feed, and that readers then become contributors.
Proof area: Vhembe. Proof window: three weeks. Proof metric: community confirmations and reports in week three.

## 1. Scope after cuts

Kept:
1. Find stream URLs for Vhembe and Limpopo stations.
2. Record streams on a schedule and keep the raw audio.
3. Transcribe audio, rough quality accepted.
4. Extract local events: type, place, time, summary, confidence.
5. Resolve place names to coordinates.
6. Enrich with official weather warnings.
7. Publish a daily brief where people already are.
8. Week three: collect one-tap confirmations and short reports.

Cut for the proof: custom app, map UI, World Monitor fork, Ushahidi deployment, push notifications, user accounts, multi-province rollout.

## 2. Reuse table

| Capability | Component | Source | Action | Licence | Risk |
|---|---|---|---|---|---|
| Stream discovery | SABC stations stream online (confirmed by Ndamu). [Radio Browser API](https://github.com/ivandotv/radio-browser-api) as a lookup aid for direct stream URLs. | Official + OSS | Resolve the direct audio URL for Phalaphala FM and one more SABC station. No code kept. | Open API | The SABC web player may wrap the stream. Resolve the underlying MP3/AAC/HLS URL before starting the monitor. |
| Record + transcribe + batch to LLM + store + live view | [MuddyWinds/Readback](https://github.com/MuddyWinds/Readback) | OSS | **Fork.** Pipeline is stream → ffmpeg chunker → faster-whisper → asyncio queue → 5-min LLM batch → Postgres → WebSocket → React. Replace the ATC prompt and the aviation side panels. | MIT | Small project (4 stars). We own the fork after day one, so this is acceptable. |
| ASR model | faster-whisper (inside Readback, `WHISPER_MODEL` env var) | Established | Adopt. Use `small` or `medium` on CPU. | MIT | **Whisper has no Tshivenda language code (assumption, verify on day one).** Run auto-detect on a Phalaphala clip. If output is useless, swap to the Meta MMS Tshivenda adapter from the ASR paper work. |
| Event extraction | Readback batcher (`backend/core/batcher.py`) | OSS fork | Replace prompt and output schema. Point the client at the self-hosted LiteLLM gateway instead of Gemini. | MIT | Prompt design is the main judgment call. |
| Place resolution | GeoNames South Africa gazetteer dump (`ZA.zip`) | Established data | LLM returns place names. A lookup table resolves them to coordinates and district. | CC BY | Village names with no GeoNames entry. Log misses, add by hand. |
| Weather enrichment | SAWS CAP RSS feed: `http://caps.weathersa.co.za/Home/RssFeed` (listed in the WMO Register of Alerting Authorities) | Official | Poll every 15 minutes. Match warnings to areas mentioned on radio. | Official feed | **SAWS warnings must be passed on as received, unaltered.** Quote the warning verbatim with a link. Never let the LLM rewrite it. |
| Daily brief distribution | Chance Africa Facebook pages via the existing facebook-pages MCP | Internal | Adopt. Post one brief per day. | Internal | Brief tone must not dilute the Chance brand. Use a clear "Vhembe Today" format. |
| Week three contributions | Facebook post comments and a pinned "confirm or report" post, read back via facebook-pages MCP | Internal | Adopt. | Internal | Low volume is the real test, not a bug. |
| Later: feed and map | [koala73/worldmonitor](https://github.com/koala73/worldmonitor) | OSS | Park. Revisit after the proof. | AGPL-3.0-only. Commercial use allowed, but a modified public deployment must publish its source. Name and logo need separate permission. | AGPL obligations on a public fork. |
| Later: structured community reports | Ushahidi Platform (Laravel, REST API v5, WhatsApp arrives as SMS source) | OSS | Park. Adopt only if week-three volume justifies moderation tooling. | AGPL | PHP stack outside our usual one. |

Internal sweep: repowise index is 36 days behind HEAD with a mock embedder, so it returned nothing useful. The Chance Africa scrapers on the apps VM are the closest internal pattern for scheduling and LLM calls. Re-index before step 4 if the agent wants to copy their LLM client.

Rejected:
- Mordecai geoparser: needs Elasticsearch and English-trained models. An LLM plus a gazetteer lookup is cheaper.
- rradio and cron-streamripper: Readback already ingests streams with ffmpeg. A second recorder adds infrastructure for no gain.

## 3. Build-only items

| Item | Why no reuse | Size |
|---|---|---|
| Raw audio retention in the fork (ffmpeg segment output to disk, hourly files) | Readback discards chunks after transcription. We need them for the Tshivenda corpus. | S |
| Local-event extraction prompt and JSON schema | Specific to this product. | M |
| Gazetteer lookup (GeoNames ZA to a small Postgres table, fuzzy match) | Trivial glue. | S |
| SAWS feed poller and area matcher | Trivial glue. | S |
| Daily brief generator (events of last 24h + SAWS warnings → post text) | Specific to this product. | S |
| Strip aviation code (ADS-B, METAR, NOTAM, HFACS) from the fork | Required cleanup. | S |

## 4. Execution order

1. **Station check.** Owner: Claude. Resolve direct stream URLs for Phalaphala FM and one other SABC station. Probe each for 30 seconds with ffprobe. Done when: two working stream URLs are recorded in `stations.yml`.
2. **ASR spike.** Owner: Claude. Record 10 minutes of Phalaphala news. Run faster-whisper `medium` with auto-detect. Done when: transcript saved, and a frontier model can state what the bulletin covered. If not, note it and queue the MMS swap.
3. **Fork and strip.** Owner: Codex. Fork exists at `ndamulelonemakh/Readback`. Remove aviation modules. Done when: `docker compose up` starts, tests pass, one radio feed transcribes into Postgres.
4. **Extraction.** Owner: Claude for the prompt and schema, Codex for wiring. Replace the batcher prompt. Route calls through LiteLLM. Done when: a one-hour recording yields an events table with type, place, time, summary, confidence.
5. **Place and weather.** Owner: Copilot or Codex. Load GeoNames ZA. Add the SAWS poller. Done when: events carry coordinates, and an active SAWS warning for Limpopo attaches to matching events.
6. **Audio retention.** Owner: Codex. Keep hourly raw audio files. Done when: files land in `~/data/radio/<station>/<date>/` with a manifest.
7. **Brief.** Owner: Claude. Generate the daily brief and post it through the facebook-pages MCP. Done when: one brief is posted after Ndamu approves the first one.
8. **Week three.** Owner: Claude. Pin the "confirm or report" post. Count replies daily. Done when: a short results note goes into Notion under Rooral HQ.

Read rules for agents:
- Read: `backend/core/`, `backend/main.py`, `docker-compose.yml`, `.env.example`, `README.md`.
- Do not read: `frontend/node_modules`, lockfiles, `tests/fixtures`, aviation data tables.

Deployment notes for the apps VM:
- The VM already runs native PostgreSQL 17 on 5432. Readback's compose also binds 5432. Either point the fork at the host Postgres with a new database, or remap the container port.
- Check that 3000 and 8000 are free against the pm2 apps before starting. Put the dashboard behind Caddy on a tailnet-only route.

## 5. Demo harness

Inputs: two stream URLs, a two-hour recording window, the current SAWS feed.

Commands:
```
docker compose up -d
curl -X POST "http://localhost:8000/api/monitor/start?feed_url=<stream>&airport_code=PHALAPHALA"
# after two hours
curl http://localhost:8000/api/results
```
(Rename `airport_code` to `station_code` during step 3. Shown here as Readback ships it.)

Expected outputs:
- Transcripts in Postgres for both stations.
- At least five extracted events with a resolved place in Vhembe.
- Hourly audio files on disk.
- One draft brief in plain text, ready for approval.

## 6. Cost estimate

- Agent time: two to three focused sessions for steps 1 to 6. Step 4 carries most of the frontier-model use.
- LLM tokens: one batch call every five minutes per station. Two stations for 12 hours a day is about 290 calls a day. Cheap model through LiteLLM, low rands per day.
- Compute: CPU faster-whisper `medium` on the apps VM is the main load. If it falls behind real time, drop to `small` or record only news bulletins and talk shows.
- Storage: compressed mono audio at 32 to 64 kbps is roughly 0.3 to 0.6 GB per station per day. Check the apps VM disk before step 6.

## 7. Assumptions

- Whisper does not support Tshivenda natively (from memory, verify in step 2).
- Phalaphala streams online through SABC. Radio Browser coverage of SA community stations was not checked station by station.
- Summarising broadcasts into a short brief with attribution to the station is acceptable for the proof. Raw audio is kept for research and is not republished.
- The Chance Africa Facebook page is the first distribution channel.
- Vhembe is the proof area.
