# Claude Code Skills Collection

[![Works with Claude Code](https://img.shields.io/badge/Works%20with-Claude%20Code-blueviolet)](https://claude.com/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

12 production-tested skills for [Claude Code](https://claude.com/claude-code) CLI — covering SEO, marketing, content creation, publishing, and DevOps.

## Skills

| Skill | Description |
|-------|-------------|
| **agents** | Design, build, and debug AI agent systems — multi-step reasoning, tool use, memory, orchestration, and autonomous workflows |
| **content-creation** | Create high-quality content for blogs, landing pages, email sequences, social media, video scripts, newsletters, and more |
| **content-pipeline** | Multi-agent content production pipeline — research, write, edit, optimize for SEO/GEO, and validate |
| **context-engineering** | Manage, optimize, and debug Claude Code context windows and autocompaction |
| **distribution** | Multi-platform digital product distribution — track accounts, format descriptions, manage cross-links, coordinate publishing |
| **ebook-publishing** | Complete ebook and audiobook self-publishing — write, format, convert, and publish across all major platforms |
| **marketing** | Plan, write, and optimize marketing campaigns, funnels, ad copy, personas, positioning, and go-to-market strategy |
| **memory-persistence** | Implement cross-session memory, persistent agent knowledge, and session continuity for Claude Code |
| **seo-geo** | Complete SEO and GEO (Generative Engine Optimization) — keyword research, SERP analysis, technical audits, content optimization, backlink strategy |
| **token-optimizer** | Reduce Claude Code token consumption and API costs without sacrificing output quality |
| **vps-ubuntu** | Manage, secure, monitor, and maintain Ubuntu VPS servers |
| **wordpress** | Publish, manage, and optimize WordPress content via the REST API — posts, pages, media, Gutenberg blocks, WooCommerce |

## Installation

Copy any skill folder into your Claude Code skills directory:

```bash
# Install a single skill
cp -r skills/seo-geo ~/.claude/skills/seo-geo

# Install all skills
cp -r skills/* ~/.claude/skills/
```

Skills auto-trigger based on conversation context — no manual activation needed. When you discuss a topic that matches a skill's domain, Claude Code will automatically load and apply the relevant skill.

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI

## Author

[The GEO Lab](https://thegeolab.net)

## License

[MIT](LICENSE)
