# GEO Compliance Checklist (25 Checks)

Reference for auditing content against GEO compliance standards. Each check includes verification method and bash/grep patterns for automated testing.

## Schema Checks (5)

| Check | Requirement | How to Verify | Pass Threshold | Bash Command |
|-------|-------------|---------------|-----------------|--------------|
| FAQPage Schema | BlogPost/Article has FAQPage if ≥3 questions | Inspect JSON-LD block; check `@type` | Present + valid JSON | `curl -s https://URL \| grep -o '"@type":"FAQPage"'` |
| Article/BlogPosting | Main content type correct for page | Check `@type` is Article or BlogPosting (not generic WebPage) | Must be present | `curl -s https://URL \| grep -o '"@type":"\\(Article\\|BlogPosting\\)"'` |
| Organization Schema | Homepage has org schema with name, logo, contact | JSON-LD `@type: Organization` on home | Present + logo + contact | `curl -s https://thegeolab.net \| grep -oP '"@type":"Organization".*?"name":"[^"]*"'` |
| Review Schema (≥2) | Product/Service pages have ≥2 Review objects | Inspect aggregateRating in schema | ≥2 reviews minimum | `curl -s https://URL \| grep -co '"@type":"Review"'` |
| Breadcrumb Schema | All pages >2 levels deep have breadcrumb | JSON-LD `@type: BreadcrumbList` | Present for nested pages | `curl -s https://URL \| grep -o '"@type":"BreadcrumbList"'` |

## Structure Checks (8)

| Check | Requirement | How to Verify | Pass Threshold | Bash Command |
|-------|-------------|---------------|-----------------|--------------|
| Brand in Title | Page title contains "GEO Lab" or relevant brand | HTML `<title>` tag | Must contain brand | `curl -s https://URL \| grep -o '<title>[^<]*GEO[^<]*</title>'` |
| Single H1 | Exactly one H1 tag per page | Count H1 tags in rendered HTML | Exactly 1 | `curl -s https://URL \| grep -co '<h1[^>]*>'` |
| TL;DR Present | Post begins with bolded TL;DR or summary | Check first 500 chars for bold summary | Present in first 2 paras | `curl -s https://URL \| grep -o '<p><strong>TL;DR'` |
| FAQ Section | Blog posts ≥2000 words have FAQ section | Grep for "FAQ\|Frequently" in H2/H3 | Present for longer content | `curl -s https://URL \| grep -io '<h[2-3][^>]*>.*FAQ'` |
| Table of Contents | Posts ≥2500 words have TOC | Check for `nav` or `class="toc"` | Present for longer content | `curl -s https://URL \| grep -o 'class="toc"'` |
| Figure Tag Usage | Posts have ≥1 `<figure>` tag for images | Parse HTML for `<figure>` elements | ≥1 minimum | `curl -s https://URL \| grep -co '<figure'` |
| Key Takeaways | Posts ≥1500 words have Key Takeaways section | Grep for "Key Takeaways\|Takeaway" in H2/H3 | Present for longer posts | `curl -s https://URL \| grep -io '<h[2-3][^>]*>.*Takeaway'` |
| Version History | Updated posts have version date in meta | Check post_modified field in DB or footer | Must match update | `wp post get 123 --field=post_modified` |

## Voice Checks (5)

| Check | Requirement | How to Verify | Pass Threshold | Bash Command |
|-------|-------------|---------------|-----------------|--------------|
| Author Bio Present | Post has byline with author name/role | Check WP author field + bio block | Must be set + visible | `wp post get 123 --field=post_author` |
| "Since 2004" Reference | About/credibility content mentions founding year | Grep for "since 2004\|founded 2004" | ≥1 occurrence | `curl -s https://thegeolab.net/about \| grep -i 'since 2004'` |
| Contact Link Present | Post includes visible contact CTA link | Grep for contact page link | ≥1 in header/footer/body | `curl -s https://URL \| grep -o 'href="[^"]*contact[^"]*"'` |
| First-Person I Count | Post uses first-person pronoun ≥2 times | Count "I " (word boundary) in body | ≥2 occurrences | `curl -s https://URL \| grep -io ' i ' \| wc -l` |
| No Self-Review | Post does not review own products/services | Check author is not reviewer; no self-citations | Zero self-citations | grep author_id = reviewer_id from DB |

## Link Checks (2)

| Check | Requirement | How to Verify | Pass Threshold | Bash Command |
|-------|-------------|---------------|-----------------|--------------|
| Internal Links ≥4 | Post links to ≥4 other site pages | Extract `href` links pointing to same domain | ≥4 minimum | `curl -s https://URL \| grep -o 'href="https://thegeolab.net[^"]*"' \| sort -u \| wc -l` |
| External Links ≥2 | Post cites ≥2 external authoritative sources | Extract `href` links to external domains | ≥2 minimum | `curl -s https://URL \| grep -o 'href="https://[^thegeolab][^"]*"' \| sort -u \| wc -l` |

## New Checks (5)

| Check | Requirement | How to Verify | Pass Threshold | Bash Command |
|-------|-------------|---------------|-----------------|--------------|
| Closing CTA | Post ends with explicit call-to-action | Check last 300 chars for CTA verb (Read, Download, Join, Subscribe) | ≥1 CTA in closing | `curl -s https://URL \| tail -c 300 \| grep -iE '(Read|Download|Join|Subscribe)'` |
| Ebook Callout | Long-form content (≥3000w) mentions related ebook | Grep for ebook callout box or link | Present if applicable | `curl -s https://URL \| grep -io 'ebook\|download.*guide'` |
| Citation-Ready Claims ≥1 | Post contains ≥1 claim with verifiable source/stat | Check for (stat, study, quote) with attribution | ≥1 minimum | `curl -s https://URL \| grep -E '\([0-9]+%\|study\|research\|found that\)'` |
| Stat Currency | Stats are dated or cited with source year | All numeric claims include year or "recent" label | No undated stats | `curl -s https://URL \| grep -o '[0-9]\+%' \| wc -l` (manual review) |

## Compliance Pass/Fail Rules

- **Pass**: ≥22/25 checks passing (88% threshold)
- **Fail**: <22/25; flag category with most failures for revision
- **Critical fail**: Any schema check fails + any brand check fails = immediate rejection

## Automated Compliance Report

```bash
#!/bin/bash
URL=$1
echo "=== COMPLIANCE AUDIT: $URL ==="

# Schema (5)
echo "Schema Checks:"
curl -s "$URL" | grep -q '"@type":"FAQPage"' && echo "✓ FAQPage" || echo "✗ FAQPage"
curl -s "$URL" | grep -q '"@type":"Article"' && echo "✓ Article" || echo "✗ Article"
curl -s "$URL" | grep -q '"@type":"Organization"' && echo "✓ Organization" || echo "✗ Organization"
curl -s "$URL" | grep -c '"@type":"Review"' | awk '{if ($1>=2) print "✓ Reviews"; else print "✗ Reviews"}'
curl -s "$URL" | grep -q '"@type":"BreadcrumbList"' && echo "✓ Breadcrumb" || echo "✗ Breadcrumb"

# Structure (8)
echo -e "\nStructure Checks:"
curl -s "$URL" | grep -q '<title>.*GEO' && echo "✓ Brand Title" || echo "✗ Brand Title"
H1_COUNT=$(curl -s "$URL" | grep -co '<h1[^>]*>')
[ "$H1_COUNT" -eq 1 ] && echo "✓ Single H1" || echo "✗ Single H1 (found $H1_COUNT)"
curl -s "$URL" | grep -q '<p><strong>TL;DR' && echo "✓ TL;DR" || echo "✗ TL;DR"

# Voice (5)
echo -e "\nVoice Checks:"
curl -s "$URL" | grep -q 'class="author' && echo "✓ Author Bio" || echo "✗ Author Bio"
curl -s "$URL" | grep -iq 'since 2004' && echo "✓ Since 2004" || echo "✗ Since 2004"

# Links (2)
echo -e "\nLink Checks:"
INTERNAL=$(curl -s "$URL" | grep -co 'href="https://thegeolab.net')
[ "$INTERNAL" -ge 4 ] && echo "✓ Internal Links (n=$INTERNAL)" || echo "✗ Internal Links (n=$INTERNAL)"
EXTERNAL=$(curl -s "$URL" | grep 'href="https://' | grep -v 'thegeolab.net' | wc -l)
[ "$EXTERNAL" -ge 2 ] && echo "✓ External Links (n=$EXTERNAL)" || echo "✗ External Links (n=$EXTERNAL)"
```
