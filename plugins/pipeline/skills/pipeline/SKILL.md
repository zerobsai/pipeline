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

## Habits

- When you finish producing something (draft, audit, creative), `attach_artifact` it and `advance_stage` the item in the same breath — Pipeline is only useful if state lands there.
- When you create derived content (a LinkedIn post from a blog article), `link_derivative` immediately.
- Check `list_open_findings` before starting new content for a site — fixing an overdue critical finding usually beats writing a new post.
