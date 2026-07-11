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

The server also proxies seven paid third-party APIs behind Pipeline's OAuth, using platform keys metered per org — no local SERP/keyword/image keys to configure. Reach for these as the zero-setup path; if the user already has their own local keys (`GOOGLE_AI_API_KEY`, a DataForSEO account, etc.), those are the first choice and these are the fallback.

| Task | Tool |
|---|---|
| Google SERP (organic, people-also-ask, related searches) for competitor discovery, fact-finding | `serp_search` (query, country?, language?, num?) |
| Real monthly volume/CPC/competition for up to 100 keywords, or ideas expanded from a seed | `keyword_research` (mode `volume` default or `ideas`, keywords?, seed?, location_code?, language_code?) |
| Any URL → clean markdown (JS-rendered included) for competitor content analysis | `scrape_url` (url) |
| PageSpeed: category scores + lab Core Web Vitals + field data | `pagespeed_check` (url, strategy?) |
| Generate an image (Gemini "Nano Banana") stored as a Pipeline artifact | `generate_image` (prompt, aspect_ratio?, content_item_id?, filename?, kind?) → returns `artifact_id` |
| Download an artifact's bytes as base64 | `get_artifact_content` (artifact_id) |
| Per-tool usage totals for the active org | `get_ai_usage` (days?) |

- **Not enabled?** Any tool whose provider key isn't configured on the server returns a clear "not enabled on this Pipeline server" error. Treat that as a signal to fall back to the user's local key/tooling, not to retry.
- **Image → local file flow.** `generate_image` doesn't hand back bytes; it stores the image as an artifact (`kind` defaults to `hero`, optionally linked via `content_item_id`) and returns `artifact_id`. To land it on disk (e.g. the hero image a blog delivery contract expects), call `get_artifact_content(artifact_id)` and write the decoded base64 to the file.

## Habits

- When you finish producing something (draft, audit, creative), `attach_artifact` it and `advance_stage` the item in the same breath — Pipeline is only useful if state lands there.
- When you create derived content (a LinkedIn post from a blog article), `link_derivative` immediately.
- Check `list_open_findings` before starting new content for a site — fixing an overdue critical finding usually beats writing a new post.
