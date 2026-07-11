# Pipeline — Claude Code plugin

Connect Claude to [Pipeline](https://pipeline.zerobsai.com), the system of record for your marketing content: a stage-based content pipeline with brand kits, audits with a finding lifecycle, a content calendar, a repurposing graph, and hosted AI-workflow tools (SERP search, keyword research, URL scraping, PageSpeed, image/video/audio/music generation, server-side reel assembly, competitor tracking and content gaps, semantic search over your content) that run on platform keys so no local API setup is needed — subscription or bring-your-own keys — exposed to Claude as an OAuth-protected MCP server.

It's designed to pair with the stateless marketing skill suites, giving them shared, persistent state:

- [claude-blog](https://github.com/AgriciDaniel/claude-blog) — blog production
- [claude-seo](https://github.com/AgriciDaniel/claude-seo) — SEO audits
- [claude-ads](https://github.com/AgriciDaniel/claude-ads) — paid-ads audits

## Install

```
/plugin marketplace add zerobsai/pipeline
/plugin install pipeline@zerobsai
```

That registers the remote MCP server (no API keys — OAuth in the browser) and three skills:

| Skill | Purpose |
|---|---|
| `pipeline-setup` | Blank-slate onboarding: account, OAuth, org/brand, brand kit, sites, companion skills |
| `pipeline` | Day-to-day: queue, stages, artifacts, findings, calendar |
| `pipeline-sync` | Push claude-blog/seo/ads outputs into Pipeline; generate their brand files from the server-side brand kit |

## Get started

1. Sign up at https://pipeline.zerobsai.com and create an org (and brands, if you run more than one).
2. In Claude Code, run `/mcp` and authenticate the `pipeline` server.
3. Say **"set up my pipeline"** — the `pipeline-setup` skill walks the rest: activating your org/brand, building the brand kit (it can ingest existing `BRAND.md`/`VOICE.md`/`brand-profile.json` files), registering sites and ad accounts, and optionally installing the companion skills.

## How the pieces fit

The companion skills stay unmodified — they keep reading local brand files and writing local reports. This plugin makes Claude treat those as generated inputs and syncable outputs:

- **Down:** `get_brand_context` materializes the server-side brand kit into the local files each skill expects.
- **Up:** audit envelopes go to `submit_audit` (findings dedupe by fingerprint; fixed issues auto-close), drafts and reports to `attach_artifact`, repurposed content to `link_derivative`, and every post is keyed by its slug — the same universal ID claude-blog already uses.

## License

MIT
