---
name: pipeline-sync
description: Sync the claude-blog, claude-seo, and claude-ads skills with the Pipeline MCP server — submitting audits, recording content items and artifacts, and keeping brand files generated from the server-side brand kit. Use after running any of those skills, or when their outputs need to land in Pipeline.
---

# Pipeline: syncing the companion skills

The claude-blog / claude-seo / claude-ads suites are stateless — they read and write local files. Pipeline is the system of record. This skill is the glue: **never fork or edit those skills**; run them as-is, then push their machine-readable outputs into Pipeline and regenerate their brand inputs from it.

## Direction of truth

- **Brand context flows down.** `get_brand_context` → write the local files the skills expect (`BRAND.md`, `VOICE.md`, `brand-profile.json`, persona JSONs). Regenerate rather than hand-edit; if the user changes a local brand file deliberately, mirror it back with `update_brand_kit`.
- **Work products flow up.** Audits, drafts, reports, and derivation links go into Pipeline via the mappings below.
- **Slug is the shared key.** claude-blog already uses the slug as its universal ID; Pipeline does too. One slug = one content item, everywhere.

## claude-seo → Pipeline

After an SEO audit completes, it writes an `audit-data.json` envelope (health score, categories, findings, phased action plan).

1. `submit_audit` with `kind: "seo"`, the target site, and the envelope as `payload`.
2. Findings are auto-created server-side with fingerprint dedupe: re-submitting an audit **refreshes** matching open findings, and open findings of the same kind+target that are *absent* from the new submission are auto-marked **fixed** — so always submit the complete envelope, never a filtered subset.
3. Attach the human-readable report (PDF/markdown) to the audit with `attach_artifact`.
4. Between audits, work findings via `list_open_findings` / `update_finding` (status, owner; severity changes recompute the SLA due date).

## claude-ads → Pipeline

claude-ads scores accounts against a catalog of stable check IDs.

1. `submit_audit` with `kind: "ads"`, the target ad account, and `payload` as the per-check result vector: `[{check_id, result}, ...]`. Use the skill's own check IDs verbatim — they're the fingerprint.
2. Same dedupe/auto-fix lifecycle as SEO audits; same artifact attachment for the report.

## claude-blog → Pipeline

1. **Brief accepted** → `create_item` with the post's slug, title, brief, channel `blog`, and target site. Do this when the slug is decided, not at publish time — the pipeline should show work in flight.
2. **Draft/revision produced** → `attach_artifact` the markdown against the item; `advance_stage` as it moves (drafting → review → etc.). Stage moves *forward out of* a human-gate stage are refused by the server — a human clears gates in the web UI; report the gate message and stop.
3. **Preflight/review results** → the skill's `preflight-report.json` and review scorecard (the `BLOCKING:` line) are machine-readable: `submit_audit` with `kind: "blog_quality"`, target = the item's slug.
4. **Published** → `update_item` with the live URL and published date.
5. **Repurposed content** (social post, newsletter cut from an article) → `create_item` for the derivative on its channel, then `link_derivative` (child derived_from parent). Check `list_repurpose_gaps` to find published items missing declared channels — that's the repurposing to-do list.

## When to sync

Sync at the moment an output file is finalized, in the same session that produced it — don't batch. A skill run whose results only live in local files is invisible to the rest of the org.
