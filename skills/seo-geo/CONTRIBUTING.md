# Contributing to seo-geo-skill

Thanks for helping keep this skill accurate and useful. SEO APIs, tools, and best
practices change constantly — community contributions are what keep this the most
up-to-date SEO/GEO skill available.

## What We Welcome

### High-value contributions
- **API/tool updates** — endpoint changes, pricing changes, new features, deprecations
- **New failure entries** — confirmed bugs with reproduction steps and fixes
- **New reference sections** — well-researched additions to any reference file
- **Bug fixes** — incorrect commands, broken code snippets, outdated instructions
- **New data sources** — free SEO data sources worth adding to the stack

### What we don't accept
- Unverified data (guesses, approximations without sources)
- Promotional content for specific paid services
- Changes to `project-context.md` (that's brand-specific to The GeoLab)
- Breaking changes to the SKILL.md frontmatter format without discussion first

---

## How to Contribute

### 1. Check existing issues first
Someone may already be tracking the change you spotted.

### 2. Fork and create a branch
```bash
git clone https://github.com/YOUR_USERNAME/seo-geo-skill.git
cd seo-geo-skill
git checkout -b update/serper-api-changes
```

### 3. Make your changes
- Keep edits focused — one topic per PR
- Always include your source (official API docs, not blog posts)
- Update the date reference in the file if the data changed
- Add an entry to the relevant section in `README.md` changelog if significant

### 4. Submit a Pull Request
Use the PR template — it asks for source links and a brief explanation.
We review all PRs within 7 days.

---

## File Guide

| File | What it contains | How often it changes |
|---|---|---|
| `SKILL.md` | Main skill — workflow overview, decision trees | Rarely |
| `references/keyword-research.md` | Free keyword research stack | Occasionally |
| `references/serp-analysis.md` | SERP analysis via Serper.dev | Occasionally |
| `references/technical-audit.md` | Technical audit checks | Rarely |
| `references/backlinks.md` | Free backlink intelligence | Occasionally |
| `references/infrastructure.md` | Building tools — proxies, caching, deploy | Occasionally |
| `references/failures.md` | Confirmed failure registry | Frequently (grows with experience) |

**Most common update targets**: `references/failures.md` (new failures discovered),
`references/infrastructure.md` (API pricing/endpoint changes), and
`references/serp-analysis.md` (Serper.dev updates).

---

## Versioning

We follow [Semantic Versioning](https://semver.org/):
- **Patch** (1.0.x): Data corrections, typo fixes, small clarifications
- **Minor** (1.x.0): New data sources, new reference sections, new workflows
- **Major** (x.0.0): Structural changes to SKILL.md, breaking changes to file layout

---

## Code of Conduct

Be accurate, be useful, cite your sources. That's it.

---

## Questions?

Open a [Discussion](https://github.com/arturseo-geo/seo-geo-skill/discussions)
or reach out via [𝕏 @TheGEO_Lab](https://x.com/TheGEO_Lab).
