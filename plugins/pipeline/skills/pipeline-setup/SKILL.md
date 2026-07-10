---
name: pipeline-setup
description: One-time onboarding for the Pipeline marketing MCP server — account, OAuth connection, org/brand selection, brand kit, sites, and optional install of the claude-blog/claude-seo/claude-ads companion skills. Use when Pipeline tools are missing, unauthenticated, or the user asks to set up their marketing pipeline.
---

# Pipeline: setup from a blank slate

Walk these steps in order. Every step is verifiable via MCP, so this skill is safe to re-run — check state first, then only do what's missing.

## 1. Account and org (web UI — the only human-browser steps)

MCP has no tools to create accounts, orgs, or brands. If any of these don't exist yet, send the user to **https://pipeline.zerobsai.com**:

1. Sign up (email/password or Google).
2. Create an organization.
3. Create at least one brand inside it (skippable for a single-brand org — one is created with the org).

## 2. Connect the MCP server

The plugin already registered the `pipeline` MCP server (https://pipeline.zerobsai.com/api/mcp). If `mcp__pipeline__*` tools fail with an auth error or aren't listed, tell the user to run `/mcp` and authenticate the `pipeline` server — OAuth completes in the browser against the account from step 1. No API keys, no config editing.

## 3. Scope: org and brand

1. `set_active_org` — only needed if the user belongs to more than one org.
2. `set_active_brand` — required before any writes in a multi-brand org. Reads without an active brand roll up across all brands.

Verify with `get_brand_context` — it should return without error (an empty brand kit is fine at this point).

## 4. Brand kit

The brand kit is Pipeline's server-side replacement for the loose brand files the companion skills use (claude-blog's `BRAND.md`/`VOICE.md`, claude-seo's `brand-profile.json`, persona JSONs). Populate it with `update_brand_kit`, section by section — object sections shallow-merge, `personas` is replaced wholesale:

- **If local brand files exist** (search the repo for `BRAND.md`, `VOICE.md`, `brand-profile.json`, persona files): read them and map their content into the matching sections (`audience`, `positioning`, `voice`, `editorial_rules`, `topic_scope`, `personas`, `visual`, `facts`). Show the user what you're about to write before writing.
- **If nothing exists**: interview the user briefly (who is the audience, what's the positioning, voice do/don'ts, hard facts about the product) and write what you learn. A thin brand kit now beats a blank one — it grows with `update_brand_kit` later.

## 5. Sites and ad accounts

- `create_site` for each web property (blog, marketing site) — claude-seo audits target these.
- `create_ad_account` for each ad platform account — claude-ads audits target these.
- `list_sites` to confirm.

## 6. Companion skills (optional but recommended)

Pipeline is the state sink for three stateless skill suites. If the user wants them, clone each next to their working repo and follow its own README:

- **claude-blog** — https://github.com/AgriciDaniel/claude-blog (blog production, 30 sub-skills)
- **claude-seo** — https://github.com/AgriciDaniel/claude-seo (SEO audits, 25 sub-skills)
- **claude-ads** — https://github.com/AgriciDaniel/claude-ads (paid-ads audits, 250+ checks)

Then materialize the brand files they expect from Pipeline: call `get_brand_context` and write its sections into the local files each skill reads (`BRAND.md`, `VOICE.md`, `brand-profile.json`). Pipeline is the source of truth; the local files are generated copies — regenerate them when the kit changes rather than editing them by hand. The `pipeline-sync` skill covers pushing their outputs back.

## 7. Verify

- `get_brand_context` returns the kit and channels.
- `get_pipeline` returns the stage list.
- `create_item` a throwaway test item is unnecessary — don't pollute the pipeline; the two reads above prove the connection.

Setup is done. Day-to-day usage lives in the `pipeline` skill; integration with the companion skills lives in `pipeline-sync`.
