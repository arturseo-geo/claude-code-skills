# Claude Code Skills Collection

> Built by **[Artur Ferreira](https://github.com/arturseo-geo)** @ **[The GEO Lab](https://thegeolab.net)**
> [𝕏 @TheGEO_Lab](https://x.com/TheGEO_Lab) · [LinkedIn](https://linkedin.com/in/arturgeo) · [Reddit](https://www.reddit.com/user/Alternative_Teach_74/)

[![Works with Claude Code](https://img.shields.io/badge/Works%20with-Claude%20Code-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
![Skills](https://img.shields.io/badge/skills-16-orange)

16 production-tested skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI — covering SEO, marketing, content creation, publishing, DevOps, AI agents, analytics, operations, email marketing, and research. Each skill auto-triggers based on conversation context.

## Who This Is For

- **Claude Code users** who want domain expertise loaded automatically when they need it
- **Developers** building with Claude Code who want SEO, WordPress, VPS, or agent patterns
- **Content creators** who want AI-assisted writing, publishing, and distribution workflows
- **Anyone evaluating Claude Code skills** who wants a comprehensive, well-maintained collection

## Skills

| Skill | What it teaches Claude Code | Standalone repo |
|-------|---------------------------|-----------------|
| **seo-geo** | Keyword research, SERP analysis, technical audits, backlink intelligence, AI visibility | [seo-geo-skill](https://github.com/arturseo-geo/seo-geo-skill) |
| **ebook-publishing** | 11 platforms, HTML→PDF Puppeteer workflow, AI audiobooks, ISBN strategy | [ebook-publishing-skill](https://github.com/arturseo-geo/ebook-publishing-skill) |
| **wordpress** | REST API, Gutenberg, WooCommerce, RankMath/Yoast, multisite, WP-CLI | [wordpress-skill](https://github.com/arturseo-geo/wordpress-skill) |
| **agents** | ReAct, Plan-and-Execute, MCP, LangGraph, multi-agent orchestration | [agents-skill](https://github.com/arturseo-geo/agents-skill) |
| **content-pipeline** | 6-agent content production pipeline with quality gates | [content-pipeline-skill](https://github.com/arturseo-geo/content-pipeline-skill) |
| **content-creation** | Blogs, emails, social media, video scripts, repurposing workflows | [content-creation-skill](https://github.com/arturseo-geo/content-creation-skill) |
| **marketing** | Campaigns, funnels, ad copy, StoryBrand/$100M Offers frameworks | [marketing-skill](https://github.com/arturseo-geo/marketing-skill) |
| **vps-ubuntu** | Server management, Docker, Nginx, SSL, fail2ban, BorgBackup | [vps-ubuntu-skill](https://github.com/arturseo-geo/vps-ubuntu-skill) |
| **memory-persistence** | 6 memory patterns, session continuity, hooks auto-save | [memory-persistence-skill](https://github.com/arturseo-geo/memory-persistence-skill) |
| **context-engineering** | Context window management, autocompaction, token budgets | [context-engineering-skill](https://github.com/arturseo-geo/context-engineering-skill) |
| **token-optimizer** | Model routing, prompt caching, cost tracking (60-80% savings) | [token-optimizer-skill](https://github.com/arturseo-geo/token-optimizer-skill) |
| **distribution** | 8-platform ebook distribution, cross-linking, royalty comparison | [distribution-skill](https://github.com/arturseo-geo/distribution-skill) |
| **analytics-reporting** | GSC + GA4 correlation, traffic analysis, performance reports, anomaly detection | [analytics-reporting-skill](https://github.com/arturseo-geo/analytics-reporting-skill) |
| **command-center** | VPS service operations, PM2, BullMQ, Redis, health checks, debugging | [command-center-skill](https://github.com/arturseo-geo/command-center-skill) |
| **email-marketing** | ConvertKit/Beehiiv/Substack, sequences, deliverability, segmentation, A/B testing | [email-marketing-skill](https://github.com/arturseo-geo/email-marketing-skill) |
| **research** | Competitor intelligence, market sizing, trend analysis, source gathering | [research-skill](https://github.com/arturseo-geo/research-skill) |

## Installation

### Install all skills at once

```bash
git clone https://github.com/arturseo-geo/claude-code-skills.git
cp -r claude-code-skills/skills/* ~/.claude/skills/
```

### Install a single skill

```bash
cp -r claude-code-skills/skills/seo-geo ~/.claude/skills/seo-geo
```

### Install from standalone repos

Each skill is also available as a standalone repo (see table above). This is useful if you want to star, fork, or contribute to a specific skill:

```bash
git clone https://github.com/arturseo-geo/seo-geo-skill.git ~/.claude/skills/seo-geo
```

## How Skills Work

Skills auto-trigger based on conversation context — no manual activation needed. When you discuss SEO, Claude Code loads `seo-geo`. When you mention WordPress, it loads `wordpress`. Each skill has:

- **SKILL.md** — core instructions, always loaded (~100 tokens trigger)
- **references/** — deep-dive docs, loaded on demand to save context
- **commands/** — slash commands (where applicable)

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI

## About The GEO Lab

These skills were built and battle-tested while building 7 production SEO/GEO tools at [The GEO Lab](https://thegeolab.net) — a research platform focused on Generative Engine Optimisation. Every pattern, warning, and recommendation comes from real production experience.

## Contributing

PRs welcome across all skills. See individual skill repos for specific contribution guidelines.

---

Built and maintained by **[Artur Ferreira](https://github.com/arturseo-geo)** @ **[The GEO Lab](https://thegeolab.net)** · [MIT License](LICENSE)
