---
name: pipeline
description: Day-to-day use of the Pipeline marketing MCP server — checking your queue, moving content through stages, attaching artifacts, and tracking findings. Use when working with the marketing content pipeline.
---

# Pipeline: day-to-day usage

Pipeline (https://pipeline.zerobsai.com) is the system of record for marketing content. The MCP server exposes it as tools prefixed `mcp__pipeline__`. This skill assumes setup is done (see the `pipeline-setup` skill if tools fail with no org/brand).

## Session start

1. `get_brand_context` — the brand kit (audience, positioning, voice, editorial rules, personas, facts) and declared channels. Ground all marketing work in this; it replaces local BRAND.md/VOICE.md files.
2. `get_my_queue` — items assigned to the current user, plus overdue findings.

## Core concepts

- **Slug is the universal key.** Content items are identified by slug across Pipeline and the companion skills (claude-blog, claude-seo, claude-ads). Never invent parallel IDs.
- **Stages** are an ordered pipeline (see `get_pipeline`). Some stages are **human gates**: you may move an item *into* or *backward out of* a gate, but never *forward* out of one — a human clears the gate in the web UI. `advance_stage` returns the gate explanation when it refuses; report it to the user and stop, don't retry.
- **Org and brand scope.** All tools operate on the active org (`set_active_org`) and brand (`set_active_brand`). In a multi-brand org, reads roll up across brands when no brand is active, but writes require an active brand.

## Common operations

| Task | Tool |
|---|---|
| See what's in flight | `get_pipeline` (filter by stage/owner/channel) |
| Scheduled content | `get_calendar` (from/to window) |
| Find an item | `search_content` (title/slug/brief substring), then `get_item` |
| New content idea/brief | `create_item` (defaults to first stage; set slug deliberately) |
| Edit an item's fields | `update_item` (never for stage changes) |
| Move through pipeline | `advance_stage` |
| Store a deliverable | `attach_artifact` (text or base64, max 2MB, tied to an item or audit) |
| Record repurposing | `link_derivative` (child derived_from parent; rejects cycles) |
| Find repurposing opportunities | `list_repurpose_gaps` (published items missing declared channels) |
| Audit results in | `submit_audit` (see the `pipeline-sync` skill) |
| Work a finding | `list_open_findings`, `update_finding` (status/severity/owner; severity recomputes SLA) |

## Hosted AI tools

The server also proxies a suite of paid third-party APIs behind Pipeline's OAuth, using platform keys metered per org — no local SERP/keyword/image keys to configure. Reach for these as the zero-setup path; if the user already has their own local keys (`GOOGLE_AI_API_KEY`, a DataForSEO account, etc.), those are the first choice and these are the fallback. Orgs can also add their own provider keys in Settings → API keys — BYO-key calls bypass metering entirely.

### Research & intel

| Task | Tool |
|---|---|
| Google SERP (organic, people-also-ask, related searches) for competitor discovery, fact-finding | `serp_search` (query, country?, language?, num?) |
| Real monthly volume/CPC/competition for up to 100 keywords, or ideas expanded from a seed | `keyword_research` (mode `volume` default or `ideas`, keywords?, seed?, location_code?, language_code?) |
| Search-interest trends over time | `keyword_trends` (keywords, ≤5) |
| Any URL → clean markdown (JS-rendered included) for competitor content analysis | `scrape_url` (url) |
| PageSpeed: category scores + lab Core Web Vitals + field data | `pagespeed_check` (url, strategy?) |
| Track (or untrack) a competitor domain | `track_competitor` (domain, remove?) |
| Keywords competitors rank for that you don't | `content_gap` (competitor_domain? — defaults to tracked competitors) |
| Embedding-backed search over your own content (auto-indexed on item writes) | `semantic_search_content` (query) |
| Internal-link suggestions for an item | `suggest_internal_links` (content_item_id) |

### Creative & media

| Task | Tool |
|---|---|
| Generate an image, stored as a Pipeline artifact | `generate_image` (prompt, provider?, aspect_ratio?, content_item_id?, filename?, kind?) → returns `artifact_id` + signed `download_url` |
| Edit an existing image artifact with a prompt (Gemini) | `edit_image` (artifact_id, prompt, …) |
| Platform-spec image variants | `resize_image` (artifact_id, preset? `og`/`instagram`/`story`/`square`/`twitter`, or width/height) |
| Branded OG card from the org's brand kit — deterministic, no image-gen credits | `og_image` (title, subtitle?, template? `gradient`/`solid`/`split`) |
| Short video / reels, defaults vertical 9:16 — **async** | `generate_video` (prompt, provider?, aspect_ratio?, duration_seconds?, negative_prompt?, content_item_id?, filename?) → returns `job_id` |
| TTS voiceover for reels or blog narration | `generate_audio` (text, voice? — default `Kore`, provider? `gemini` default WAV / `elevenlabs` premium voices mp3, content_item_id?, filename?) → signed `download_url` |
| Background music (MusicGen) — **async** | `generate_music` (prompt, duration_seconds?, …) → returns `job_id` |
| Whisper captions/transcript from an audio or video artifact | `transcribe_audio` (artifact_id, format? `srt`/`vtt`/`text`) |
| Assemble generated parts into a finished MP4 reel — server-side ffmpeg, synchronous ~1–2 min | `compose_reel` (video_artifact_id, voiceover_artifact_id?, music_artifact_id?, captions_artifact_id?, music_volume?, …) |

### Plumbing

| Task | Tool |
|---|---|
| Poll an async job (video, music) | `get_ai_job` (job_id) |
| Download an artifact's bytes as base64 | `get_artifact_content` (artifact_id) |
| Per-tool usage totals for the active org | `get_ai_usage` (days?) |

- **Not enabled?** Any tool whose provider key isn't configured on the server returns a clear "not enabled on this Pipeline server" error. Treat that as a signal to fall back to the user's local key/tooling, not to retry.
- **Always pass `idempotency_key`.** All seven generating tools (`generate_image`, `edit_image`, `generate_audio`, `generate_music`, `generate_video`, `transcribe_audio`, `compose_reel`) accept one — derive it from the task (e.g. slug + purpose) so retries never double-bill.
- **Billing errors.** Servers may have subscriptions enabled: "requires a subscription" / "out of AI credits" → point the user at Settings → Billing; "Rate limit exceeded" → wait a minute, then retry. Don't hammer.
- **Image providers.** `generate_image` takes `provider`: `gemini` (default, "Nano Banana"), `openai` (gpt-image-1), `stability` (SD3.5 Large), `flux` (FLUX 1.1 Pro via Replicate) — the claude-ads image-provider ladder without local keys. `kind` defaults to `hero`; link to an item via `content_item_id`.
- **Video and music are async.** `generate_video` (providers: `veo` — Google Veo 3.1 Fast, default; `kling`; `hailuo` via Replicate) and `generate_music` return `{job_id, status: "pending"}` immediately; generation takes ~1–5 minutes. Poll `get_ai_job(job_id)` every ~30s. On success the finished file is stored as an artifact (kind `video` for MP4s) and the result includes a signed `download_url` — download it straight to a local file with curl. Do **not** use `get_artifact_content` for videos (too large for base64).
- **Reel assembly flow.** `generate_video` (9:16) → `generate_audio` (voiceover) → `generate_music` → `transcribe_audio` (`srt` — reels need burned captions) → `compose_reel` to assemble the finished MP4 → download via its signed `download_url`.
- **Writing files locally.** `generate_image`, `generate_audio`, and `get_artifact_content` results include a signed `download_url` (valid 1 hour) — prefer `curl -o <file> "<url>"` over decoding base64 when landing a file on disk (e.g. the hero image a blog delivery contract expects). `get_artifact_content` still returns base64 for small files.

## Habits

- When you finish producing something (draft, audit, creative), `attach_artifact` it and `advance_stage` the item in the same breath — Pipeline is only useful if state lands there.
- When you create derived content (a LinkedIn post from a blog article), `link_derivative` immediately.
- Check `list_open_findings` before starting new content for a site — fixing an overdue critical finding usually beats writing a new post.
