---
name: distribution
description: >
  Multi-platform digital product distribution — track accounts, format
  descriptions per platform rules, manage cross-links, coordinate publishing
  schedules, and handle platform-specific gotchas. Use this skill when
  publishing products across Gumroad, Payhip, D2D, KDP, Apple Books, Kobo,
  or any combination. Also trigger when the user asks about cross-linking
  products, formatting descriptions for a specific platform, checking
  publishing status, managing tags/categories, or coordinating multi-day
  upload schedules (e.g., D2D 3/day limit).
---

# Digital Product Distribution Skill

Manage multi-platform publishing for digital products (ebooks, PDFs, tools).
Track accounts, format descriptions per platform rules, manage cross-links,
and coordinate publishing schedules.

## Reference Files

| Topic | File | Load when... |
|---|---|---|
| Platform accounts & rules | `references/platforms.md` | Publishing to any platform, formatting descriptions |
| Cross-linking strategy | `references/cross-links.md` | Updating product descriptions with links to other platforms/products |

---

## When to Use

Trigger when the user wants to:
- Publish a product across multiple platforms
- Format descriptions for a specific platform's rules
- Update cross-links between products or platforms
- Check publishing status across platforms
- Manage platform-specific metadata (tags, categories, covers)

---

## Platform Quick Reference

| Platform | Description Format | Max Length | Links Allowed | Price Floor |
|---|---|---|---|---|
| Gumroad | Plain text | No limit | Yes (in description) | $0+ (pay what you want) |
| Payhip | Plain text | 50-4,000 chars | No links, no prices, no promos | $0+ |
| D2D / Writing Life | Plain text | 50-4,000 chars | No links, no prices, no promos | $0 (free allowed) |
| Amazon KDP | Plain text (HTML in some fields) | 4,000 chars | No links | $0.99 minimum |
| Apple Books | Via D2D or direct | 4,000 chars | No links | $0 via D2D |
| Kobo | Via D2D or direct | 4,000 chars | No links | $0 via D2D |

## Upload Limits

| Platform | Daily Limit | Notes |
|---|---|---|
| Gumroad | None | Instant publish |
| Payhip | None | Instant publish |
| D2D | 3 per 24 hours | Cannot create additional accounts to bypass — account termination |
| KDP | None | Up to 72h review time |

## Cover Sizes

| Platform | Dimensions | Ratio | Notes |
|---|---|---|---|
| Amazon KDP | 1600x2560 | 5:8 | Minimum |
| Gumroad | 1280x1920 recommended | ~2:3 | 816x1302 works |
| Payhip | 2560x1600 | 8:5 | LANDSCAPE — unique |
| D2D | 1600x2400 minimum | 2:3 | |

## Description Strategy

### Platforms that allow links (Gumroad):
- Cross-link to other products in the library
- Link to website for PDF versions
- Use full domain: thegeolab.gumroad.com/l/slug

### Platforms that forbid links (Payhip, D2D, KDP):
- No URLs, emails, prices, or promotional language
- End with: "Part of the [Library Name] — N ebooks on [topic]. Written by [Author]. Published by [Publisher]."
- Keep descriptions self-contained and informative

### EPUB files:
- Add "Get the PDF version at [website URL]" link inside the EPUB itself (at end of content)
- This way the link reaches readers on all platforms without violating description rules

## Workflow

```
1. Prepare assets (EPUB, PDF, covers at multiple sizes)
2. Publish on direct platforms first (Gumroad, Payhip) — instant
3. Upload to KDP (longest review time, no daily limit)
4. Upload to D2D (3/day limit — plan across multiple days)
5. D2D distributes to Apple Books, Kobo, B&N, libraries
6. Update cross-links on Gumroad descriptions once all products live
7. For KDP permafree: report lower price from other stores after all platforms live
```
