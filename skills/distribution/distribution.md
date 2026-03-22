---
name: distribution
description: >
  Multi-platform digital product distribution — track accounts, format
  descriptions per platform rules, manage cross-links, coordinate publishing
  schedules, and handle platform-specific gotchas. Use this skill when
  publishing products across Gumroad, Payhip, D2D, KDP, Apple Books, Kobo,
  or any combination. Also trigger when the user asks about cross-linking
  products, formatting descriptions for a specific platform, checking
  publishing status, managing tags/categories, coordinating multi-day
  upload schedules (e.g., D2D 3/day limit), pricing strategy, cover
  requirements, or royalty calculations.
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
| Pricing strategies | `references/pricing.md` | Setting prices, royalty calculations, permafree strategy, KDP Select decisions |
| Cover specifications | `references/covers.md` | Preparing covers, resizing images, checking platform requirements |
| Upload scheduling | `references/scheduling.md` | Planning multi-day uploads, coordinating launch timing across platforms |

---

## When to Use

Trigger when the user wants to:
- Publish a product across multiple platforms
- Format descriptions for a specific platform's rules
- Update cross-links between products or platforms
- Check publishing status across platforms
- Manage platform-specific metadata (tags, categories, covers)
- Set or compare pricing across platforms
- Prepare cover images at correct dimensions per platform
- Plan upload scheduling around rate limits (D2D 3/day)
- Calculate royalties or compare revenue across platforms
- Decide between KDP Select (exclusive) vs. wide distribution

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
| Google Play Books | Direct or via D2D | 4,000 chars | No links | $0 (free allowed) |

## Upload Limits

| Platform | Daily Limit | Notes |
|---|---|---|
| Gumroad | None | Instant publish |
| Payhip | None | Instant publish |
| D2D | 3 per 24 hours | Cannot create additional accounts to bypass — account termination |
| KDP | None | Up to 72h review time |
| Google Play | None | Up to 48h review time |

## Cover Sizes (Quick Reference)

See `references/covers.md` for full details including DPI, format, and color space.

| Platform | Dimensions | Ratio | Notes |
|---|---|---|---|
| Amazon KDP | 1600x2560 | 5:8 | Minimum |
| Gumroad | 1280x1920 recommended | ~2:3 | 816x1302 works |
| Payhip | 2560x1600 | 8:5 | LANDSCAPE — unique |
| D2D | 1600x2400 minimum | 2:3 | |
| Apple Books | 1600x2400 | 2:3 | Via D2D |
| Kobo | 1600x2400 | 2:3 | Via D2D |
| Google Play | 1280x1920 minimum | 2:3 | |

## Pricing (Quick Reference)

See `references/pricing.md` for full royalty tiers and strategy.

| Platform | Free | Min Paid | Royalty Rate |
|---|---|---|---|
| Gumroad | $0+ | $0.99 | 90% (flat, no tier) |
| Payhip | $0+ | $0.99 | 95% (5% fee) |
| D2D | $0 | $0.99 | ~60% (varies by retailer) |
| KDP (35%) | $0.99* | $0.99 | 35% (any price) |
| KDP (70%) | N/A | $2.99 | 70% ($2.99-$9.99 range) |
| Apple Books | $0 | $0.99 | 70% (via D2D: ~60%) |
| Kobo | $0 | $0.99 | 70% (via D2D: ~60%) |

*KDP permafree requires price-match from other platforms.

## Description Strategy

### Platforms that allow links (Gumroad):
- Cross-link to other products in the library
- Link to website for PDF versions
- Use full domain: thegeolab.gumroad.com/l/slug
- Include "Also available:" section with 2-3 related products
- Mention the complete library bundle if available

### Platforms that forbid links (Payhip, D2D, KDP):
- No URLs, emails, prices, or promotional language
- End with: "Part of the [Library Name] — N ebooks on [topic]. Written by [Author]. Published by [Publisher]."
- Keep descriptions self-contained and informative
- Focus on what the reader will learn or gain
- Use bullet points for chapter highlights or key topics

### Description length targets:
- Short blurb (Gumroad card): 150-200 chars
- Full description (all platforms): 800-2,000 chars
- Never exceed 4,000 chars on any platform

### EPUB files:
- Add "Get the PDF version at [website URL]" link inside the EPUB itself (at end of content)
- This way the link reaches readers on all platforms without violating description rules

## Multi-Platform Publishing Coordination

### Pre-launch checklist:
1. EPUB passes EPUBCheck 3.0 validation
2. Cover images exported at all required sizes (see `references/covers.md`)
3. Descriptions written in two versions: with-links (Gumroad) and no-links (all others)
4. ISBN obtained if needed (D2D provides free ones)
5. Categories/BISAC codes selected
6. Keywords prepared (7 for KDP, tags for Gumroad)
7. Pricing set per platform (see `references/pricing.md`)

### Launch sequence:
1. Gumroad + Payhip first (instant publish, no review)
2. KDP next (longest review, up to 72h)
3. D2D last (3/day limit, plan across days — see `references/scheduling.md`)
4. After all platforms live: update Gumroad descriptions with cross-links
5. For KDP permafree: report lower price from other stores

### Post-launch:
- Verify all links work on each platform
- Update cross-links in `references/cross-links.md`
- Check D2D distribution status (Apple, Kobo, B&N can take 1-2 weeks)
- Monitor KDP for price-match if using permafree strategy
- Add products to Gumroad Discover once eligible (requires 1 paid sale + risk review)

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
