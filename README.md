# Podcast to Content Pipeline

An n8n automation that turns a new podcast episode into a full content suite — transcript, summaries, show notes, platform-native social posts, pull quotes, an SEO blog draft, and podcast metadata — and files everything into Notion.

**Status:** Proof of Concept, built and validated. Not yet run end to end (see [What is untested](#what-is-untested)).

---

## Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Node-by-node breakdown](#node-by-node-breakdown)
- [Setup](#setup)
- [Docker deployment](#docker-deployment)
- [Environment variables](#environment-variables)
- [Design decisions and rationale](#design-decisions-and-rationale)
- [Cost per episode](#cost-per-episode)
- [Error handling](#error-handling)
- [What is untested](#what-is-untested)
- [Known limitations](#known-limitations)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)
- [Future enhancements](#future-enhancements)

---

## What it does

```
New episode in RSS feed
        ↓
Normalize + validate episode metadata
        ↓
AssemblyAI transcription (diarization, chapters, confidence)
        ↓
Three parallel-purpose GPT-5.4 chains
   ├─ Summaries + show notes + topics + resources + action items
   ├─ LinkedIn + X + Facebook + Instagram + 10 pull quotes
   └─ SEO blog draft + keywords + tags + metadata
        ↓
Notion page (properties + body blocks)
        ↓
Slack notification with stats and a link
```

### Assets generated per episode

| Asset | Spec |
| --- | --- |
| Episode summary | 240–260 words of prose |
| Executive summary | Exactly 5 substantive sentences |
| Show notes | Markdown with real MM:SS timestamps from AssemblyAI chapters |
| Key topics | 4–8 tags |
| Resources mentioned | Books, tools, people, URLs actually referenced |
| Action items | Imperative, listener-actionable |
| Key takeaways | Durable lessons, distinct from action items |
| LinkedIn post | 1200–1800 chars, ≤3 hashtags |
| X thread | 5–8 tweets, each <280 chars |
| Facebook post | 400–700 chars, story-led |
| Instagram caption | 800–1500 chars, hook in first 125, 8–12 hashtags |
| Pull quotes | Exactly 10, verbatim, 8–25 words, graphic-ready |
| Blog draft | 900–1400 words markdown, H2 structure |
| Keywords | 2–4 primary, 5–10 secondary |
| Podcast metadata | SEO title, URL slug, meta description, categories, tags |

---

## Architecture

### Resources to create

Import both workflows, create your own Notion database and Slack channel, then replace the REPLACE_WITH_... placeholders in the main workflow with your own ids. Set the error handler as the main workflow's error workflow.

### Execution flow

```
New Podcast Episode (RSS, hourly) ─┐
                                    ├→ Normalize Episode → Episode Has Audio?
Manual Test Run → Load Sample ─────┘                          │
                                                    ┌─────────┴─────────┐
                                                  true                false
                                                    │                   │
                                        Submit Transcription Job   Alert Invalid Episode
                                                    │
                                          Wait For Transcription (45s) ←──┐
                                                    │                      │
                                          Poll Transcription Status        │
                                                    │                      │
                                             Transcription State?          │
                                    ┌───────────────┼──────────────────────┘
                                completed         error              (still processing)
                                    │               │
                        Extract Transcript Data   Alert Pipeline Failure
                                    │
                        Generate Summaries and Show Notes
                                    │
                        Generate Social Content
                                    │
                        Generate SEO and Blog
                                    │
                        Assemble Notion Payload
                                    │
                        Create Notion Episode Page
                                    │
                        Notify Content Ready
```

---

## Node-by-node breakdown

### 1. Ingest

**`New Podcast Episode`** — RSS Feed Trigger v1, polls hourly.
`feedUrl` is a **placeholder** (`https://feeds.simplecast.com/54nAGcIl`). Replace with the client feed.

**`Manual Test Run`** + **`Load Sample Episode`** — a manual trigger paired with a Set node that emits an RSS-shaped payload pointing at a real public AssemblyAI test file. Lets you exercise the whole pipeline without waiting for a new episode. Both paths converge on the same normalizer, so there is exactly one code path to maintain.

**`Normalize Episode`** — Set v3.5. Flattens either payload shape into a stable contract:

| Field | Expression |
| --- | --- |
| `episodeTitle` | `{{ $json.title ?? "Untitled Episode" }}` |
| `episodeDescription` | `{{ ($json.contentSnippet ?? $json.content ?? "").toString().slice(0, 4000) }}` |
| `publishDate` | `{{ $json.isoDate ?? $json.pubDate ?? $now.toISO() }}` |
| `audioUrl` | `{{ $json.enclosure?.url ?? $json.enclosure_url ?? "" }}` |
| `episodeGuid` | `{{ $json.guid ?? $json.link ?? $execution.id }}` |
| `audioFormat` | `{{ ($json.enclosure?.url ?? "").endsWith(".m4a") ? "m4a" : "mp3" }}` |
| `startedAt` | `{{ $now.toISO() }}` — used to compute processing time |

Every field uses optional chaining with a fallback. A malformed feed entry produces empty strings, not a crash.

**`Episode Has Audio?`** — IF v2.3. Requires `audioUrl` to match `^https?://` **and** a non-empty title. Fails closed: no audio URL means no transcription spend.

### 2. Transcribe

**`Submit Transcription Job`** — HTTP Request v4.4, `POST https://api.assemblyai.com/v2/transcript`

```json
{
  "audio_url": "{{ $json.audioUrl }}",
  "speaker_labels": true,
  "auto_chapters": true,
  "punctuate": true,
  "format_text": true,
  "language_detection": true
}
```

Auth: `genericCredentialType` → `httpHeaderAuth`. Retries 3× with 3s backoff, 30s timeout, error output wired.

**`Wait For Transcription`** — Wait v1.1, 45 seconds.

**`Poll Transcription Status`** — `GET .../v2/transcript/{{ $("Submit Transcription Job").item.json.id }}`. Retries 3× with 5s backoff.

**`Transcription State?`** — Switch v3.4:
- Output 0 `completed` → continue
- Output 1 `error` → alert
- Output 2 fallback `still processing` → **loops back to Wait**

The loop means episode length is handled by taking more laps, not by a longer timeout.

**`Extract Transcript Data`** — Set v3.5, derives:

| Field | Source |
| --- | --- |
| `transcript` | `$json.text` |
| `wordCount` | `split(/\s+/).filter(w => w).length` |
| `confidence` | rounded to 3 dp |
| `durationMin` | `audio_duration / 60` |
| `speakerCount` | `[...new Set(utterances.map(u => u.speaker))].length` |
| `chapterOutline` | chapters formatted as `MM:SS - headline - gist` |

`chapterOutline` is the key trick: it gives the LLM real timestamps in ~500 tokens instead of feeding thousands of word-level timings.

### 3. Generate

**`GPT 5.4 Content Model`** — one `lmChatOpenAi` v1.3 node shared by all three chains. Temperature 0.7, 180s timeout, 3 retries.

Three `chainLlm` v1.9 nodes, each with its own **Structured Output Parser** so downstream nodes read typed JSON rather than parsing prose:

| Chain | Schema keys |
| --- | --- |
| `Generate Summaries and Show Notes` | `summary_250`, `executive_summary[5]`, `show_notes`, `key_topics[]`, `resources_mentioned[]`, `action_items[]`, `key_takeaways[]` |
| `Generate Social Content` | `linkedin_post`, `x_thread[]`, `facebook_post`, `instagram_caption`, `pull_quotes[10]` |
| `Generate SEO and Blog` | `blog_title`, `blog_draft`, `primary_keywords[]`, `secondary_keywords[]`, `tags[]`, `seo_title`, `url_slug`, `meta_description`, `recommended_categories[]` |

Each carries a full production system prompt with explicit length targets, structural rules, and an anti-hallucination instruction ("Work only from the transcript. Never invent facts... return an empty array rather than fabricating entries").

Chains 2 and 3 read the transcript via `$("Extract Transcript Data").item.json.transcript` rather than `$json`, because a chain's output replaces `$json` with `{ output: {...} }`.

### 4. Persist and notify

**`Assemble Notion Payload`** — Set v3.5. Flattens all three chain outputs, joins arrays into strings, and **defensively truncates every property-bound field to 1900 chars**. Computes `processingSeconds` from `startedAt`.

**`Create Notion Episode Page`** — Notion v2.2, `databasePage:create`. 27 properties plus page-body blocks. Retries 3×, error output wired.

**`Notify Content Ready`** — Slack v2.5 to `#contentrepurposing` with duration, word count, speaker count, confidence %, processing time, asset list, and a Notion deep link.

---

## Setup

### 1. Credentials required

Two credentials **do not exist yet** and must be created in the n8n UI before the pipeline can run:

| Credential | Type | Where |
| --- | --- | --- |
| **AssemblyAI** | `httpHeaderAuth` | Name: `Authorization`, Value: your raw AssemblyAI API key (**no** `Bearer ` prefix) |
| **Notion** | `notionApi` | Internal integration token from notion.so/my-integrations |

Already connected and auto-assigned: **OpenAi account** (`openAiApi`).
Already connected but needs attaching to the three Slack nodes: **Slack account** (`slackOAuth2Api`).

### 2. Share the Notion database with your integration

Notion integrations see nothing by default. Open the **Podcast Content Pipeline** database → `···` → **Connections** → add your integration. Skipping this produces `object_not_found` even with a valid token.

### 3. Point the RSS trigger at the real feed

Open `New Podcast Episode` and replace `feedUrl`.

### 4. Arm the error handler

Main workflow → **Settings** → **Error Workflow** → `Podcast Pipeline Error Handler`. Without this the handler never fires.

### 5. Set the podcast name

`Normalize Episode` → `podcastName` is hardcoded to `Placeholder Podcast`.

---

## Docker deployment

See [`docker/docker-compose.yml`](docker/docker-compose.yml) and [`docker/.env.example`](docker/.env.example).

```bash
cd docker
cp .env.example .env    # fill in secrets
docker compose up -d
```

The compose file runs n8n with Postgres and enables **filesystem binary mode**, which matters if you later add an audio-download path.

**Queue mode:** for more than ~10 concurrent episodes, switch `EXECUTIONS_MODE` to `queue` and add Redis plus worker replicas. The pipeline is already compatible — it holds no in-memory state between nodes.

---

## Environment variables

| Variable | Purpose |
| --- | --- |
| `N8N_ENCRYPTION_KEY` | Encrypts stored credentials. **Losing this makes every credential unreadable.** |
| `N8N_HOST` / `WEBHOOK_URL` | Public hostname; required for correct webhook URLs |
| `DB_POSTGRESDB_*` | Postgres connection |
| `GENERIC_TIMEZONE` | Affects `$now` and cron. Set to the podcast's publishing timezone |
| `N8N_DEFAULT_BINARY_DATA_MODE` | `filesystem` recommended |
| `EXECUTIONS_DATA_PRUNE` / `_MAX_AGE` | Transcripts are large; prune or the DB grows fast |

API keys are **not** environment variables here — they live in n8n's encrypted credential store, referenced by ID. No secret appears in any node parameter.

---

## Design decisions and rationale

### Why AssemblyAI over OpenAI Whisper

This was the decisive call. **Whisper's API caps uploads at 25 MB.** A 60-minute episode at 128 kbps is roughly 57 MB — it would fail on most real episodes. Whisper also has no speaker diarization.

AssemblyAI accepts an **audio URL**, so the pipeline never downloads the file at all. That single fact removes:

- the download node and its bandwidth
- binary data handling and the memory ceiling
- any file size limit
- the MP3-vs-M4A branching the brief anticipated (AssemblyAI accepts both, plus WAV, FLAC, OGG)

It also returns diarization, per-word timestamps, and confidence scores natively — all three explicitly requested.

**Cost of this choice:** a new credential, and an async submit/poll pattern instead of one blocking call.

**Alternatives considered:**
- **Google Gemini** — native node, existing credential, handles long audio. But timestamps and speaker labels come from prompting rather than a purpose-built model, and no confidence scores. Cheapest option; a reasonable downgrade if budget dominates.
- **Deepgram** — comparable to AssemblyAI, marginally cheaper, faster. No n8n node either, so same HTTP work. Worth benchmarking if volume grows.
- **Self-hosted `whisper.cpp`** — near-zero marginal cost, no size limit, full data control. Needs a GPU box and an Execute Command node. The right move past roughly 200 episodes/month.

### Why three LLM chains instead of one

A single mega-prompt producing all 15 assets would be one API call and cheaper. Rejected because:

1. **Failure isolation** — if social generation fails, summaries and SEO still land. One call means one failure loses everything.
2. **Schema tightness** — three focused schemas validate more reliably than one 15-key schema.
3. **Output length** — one call producing a 1400-word blog *plus* everything else risks truncation.
4. **Modularity** — you can swap the social chain for a cheaper model without touching the others.

They run sequentially rather than in parallel, deliberately: parallel branches would need a 3-input Merge node, and n8n's Merge defaults to 2 inputs — the third connection silently drops. Sequential chaining with explicit `$("Node")` references avoids that failure mode entirely. Latency cost is roughly 40–60 seconds.

### Why `chainLlm` and not an AI Agent

Nothing here needs tool use or multi-step reasoning. It is deterministic transformation: transcript in, structured JSON out. An Agent would add loop overhead and non-determinism for zero benefit.

### Why chapters rather than raw utterances for show notes

`auto_chapters` returns pre-segmented topic boundaries with timestamps and gists — roughly 500 tokens. Word-level timings for a 60-minute episode would be tens of thousands. Same output quality, a fraction of the input cost.

### Why properties *and* page blocks in Notion

Notion caps a `rich_text` **property** at 2000 characters. Short, filterable assets (summary, social posts, slug, metadata) go in properties so the database is queryable and reportable. Long-form (show notes, blog draft, pull quotes, transcript) goes in the page body, sliced across paragraph blocks.

### Why upsert-style dedupe is *not* implemented yet

The RSS trigger only fires on genuinely new items, so duplicates are unlikely in normal operation. `Episode GUID` is stored on every row as the stable key. See [Future enhancements](#future-enhancements) for the lookup-before-create pattern.

---

## Cost per episode

Assumes a **60-minute, two-speaker episode** producing a ~9,000-word transcript (~12,000 tokens).

| Component | Rate | Cost |
| --- | --- | --- |
| AssemblyAI transcription | ~$0.12/hr (Best tier) | **$0.12** |
| Chain 1 — summaries | ~12k in / ~1.5k out | ~$0.03 |
| Chain 2 — social | ~13k in / ~2k out | ~$0.04 |
| Chain 3 — SEO/blog | ~14k in / ~2.5k out | ~$0.05 |
| Notion + Slack | free tier | $0.00 |
| **Total** | | **~$0.24 per episode** |

> LLM figures are estimates. GPT-5.4 pricing was not verified against a live price list during this build — treat the ~$0.12 AI portion as an order-of-magnitude figure and confirm before quoting a client.

**At scale:** 100 episodes/month ≈ **$24**. 1,000/month ≈ **$240**.

**Reducing it:**
- `gpt-5.4-mini` for social and SEO chains → roughly 60–70% off the AI portion
- AssemblyAI Nano tier → ~$0.04/hr, lower accuracy
- Truncate transcript input for the social chain (the summary alone is often enough) → ~30% off chain 2

**Wall-clock time:** ~3–7 minutes per episode, dominated by transcription (~1 min per 15 min of audio) and the three sequential LLM calls (~40–60s total).

---

## Error handling

### In-workflow

| Failure | Handling |
| --- | --- |
| Missing/invalid RSS audio URL | `Episode Has Audio?` routes false → Slack alert. No API spend. |
| AssemblyAI submit fails | 3 retries, 3s backoff → error output → Slack alert |
| Poll request fails | 3 retries, 5s backoff → error output → Slack alert |
| Transcription returns `status: error` | Switch output 1 → Slack alert |
| Transcription still running | Fallback output loops back to Wait — no timeout failure |
| Any LLM chain fails | 2 retries → error output → Slack alert |
| Notion write fails | 3 retries, 3s backoff → error output → Slack alert |
| Slack itself fails | `continueRegularOutput` — a broken notification never fails the run |

### Global

`Podcast Pipeline Error Handler` catches anything uncaught, buckets it by message pattern (rate limit / timeout / auth / Notion write), and posts an actionable Slack alert with a link to the failed execution. **Must be wired via workflow Settings to fire.**

---

## What is untested

Reported honestly — validation is not execution.

**Verified:**
- Both workflows pass `validate_workflow`
- All node types, parameter names, and versions came from live `get_node_types` — nothing guessed
- Model ID `gpt-5.4`, Slack channel `<your-slack-channel-id>`, and the Notion database/data-source IDs were resolved against live APIs
- The Notion database exists with all 27 properties

**Not verified — no execution has been run:**
- The AssemblyAI submit/poll loop against the real API
- Whether `auto_chapters` output maps cleanly into `chapterOutline`
- Whether all three LLM chains return schema-valid JSON on a real transcript
- The Notion write, including whether `multiSelectValue` accepts the expression-produced array (see below)
- Any Slack message
- End-to-end processing time and cost

**Why:** the AssemblyAI and Notion credentials do not exist yet. The pipeline cannot run until they are created.

**Three validator warnings, assessed:**

1. `startedAt.toDateTime` "not output by Normalize Episode" — **false positive.** `startedAt` is emitted; `.toDateTime()` is an n8n string extension the static checker doesn't model.
2. `$json.error` on `Alert Pipeline Failure` — **false positive.** That node is fed by error outputs, whose shape the checker can't infer.
3. `multiSelectValue` expected array, got string — **genuine and unresolved.** The expression `{{ $json.keyTopics.split(",").filter(t => t) }}` evaluates to an array at runtime, but the SDK types `expr()` as a string. Expected to work; **this is the single most likely thing to break on first run.** If it does, see Troubleshooting.

---

## Known limitations

1. **Transcript is stored as an excerpt (~7,600 chars).** Notion blocks cap at 2000 chars each and n8n's `blockUi` takes a static array, so the transcript is sliced across 4 paragraph blocks. A 9,000-word transcript is ~50,000 chars — most of it is not persisted. Fix documented below.
2. **No deduplication on re-run.** Manually re-running an episode creates a second Notion page.
3. **Single podcast.** `podcastName` is hardcoded.
4. **English-centric prompts.** `language_detection` is on for transcription, but the generation prompts assume English output.
5. **No cost ceiling.** A 4-hour episode transcribes and generates without a guard.

---

## Maintenance

- **Watch the first 5 executions closely.** LLM schema compliance and the Notion write are where surprises live.
- **Prune executions.** Transcripts in execution data grow the DB fast. Set `EXECUTIONS_DATA_MAX_AGE`.
- **Model drift.** `gpt-5.4` is pinned. Re-evaluate quarterly.
- **AssemblyAI API version.** Pinned to `/v2/`.
- **Re-check prompts after any model change.** Length targets are model-calibrated.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `object_not_found` from Notion | Integration not shared with the database | Database → `···` → Connections → add integration |
| Notion 400 on multi-select | `multiSelectValue` receiving a string | Change the expression to return `keyTopics.split(",")` directly, or set the field to a static array and map in a Code node |
| `Unauthorized` from AssemblyAI | `Bearer ` prefix added | AssemblyAI wants the raw key as the `Authorization` value |
| Pipeline loops forever | Transcription stuck in `processing` | Check the AssemblyAI dashboard; add a lap counter (see below) |
| LLM chain fails schema validation | Model returned prose | Enable `autoFix` on the Structured Output Parser |
| Show notes have no timestamps | `auto_chapters` returned empty | Very short episodes produce no chapters; fall back to `utterances` |
| Slack posts nothing | Credential not attached | Only OpenAI was auto-assigned; attach Slack manually to all 3 nodes |
| Notion page missing long content | Content exceeded the slice windows | Expand slices or implement block-append |

---

## Future enhancements

### Immediate (removes known limitations)

**Full transcript storage** — after `Create Notion Episode Page`, add a Code node that chunks the transcript into 1900-char pieces, then a Notion `block:append` node. Per-item execution appends each chunk in order. Removes limitation 1 entirely.

**Deduplication** — insert a Notion `databasePage:getAll` filtered on `Episode GUID` before transcription; IF a page exists, skip. Makes re-runs idempotent.

**Poll lap counter** — add a counter in the Wait loop and route to failure after N laps (e.g. 40 laps ≈ 30 min) so a stuck job cannot loop forever.

**Cost guard** — IF `durationMin > 180`, route to a manual-approval branch before spending.

### Scale

**Multi-podcast** — replace the single RSS trigger with a Schedule trigger reading a feed list from a Notion/Sheets table, then Split In Batches. `Podcast` and `Episode GUID` properties already support this.

**Batch processing** — extract the transcribe-and-generate section into a sub-workflow called via Execute Workflow, then run the parent over N episodes with concurrency control.

**Queue mode** — Redis + workers. The pipeline is already stateless between nodes.

### Capability

**Multilingual** — `language_detection` already returns the language. Branch the generation prompts on it, or add a translation step.

**Vector DB + semantic search** — embed transcript chunks into a vector store keyed by episode. Enables "which episode discussed pricing objections?" across the archive.

**RAG over the back catalogue** — with the vector store in place, add an AI Agent with a retrieval tool for cross-episode Q&A and callback suggestions ("this connects to Ep 17").

**Auto-publishing** — the social assets are drafted but not posted. LinkedIn, Facebook, and Instagram credentials already exist. Gate behind manual approval on v1.

**Audiogram generation** — pull quotes plus word timestamps give exact clip boundaries for short-form video.

**Self-hosted transcription** — `whisper.cpp` or `faster-whisper` on a GPU box. Breaks even around 200 episodes/month and removes the per-minute cost entirely.

---

## Project structure

```
podcast-content-pipeline/
├── README.md                    this file
├── docker/
│   ├── docker-compose.yml       n8n + Postgres, production settings
│   └── .env.example             every variable, documented
└── workflows/
    └── README.md                how to export/import the workflow JSON
```
