# Cover Specifications by Platform

## Required Dimensions

| Platform | Min Width | Min Height | Aspect Ratio | Max File Size | Format |
|---|---|---|---|---|---|
| Amazon KDP | 1600px | 2560px | 5:8 (1:1.6) | 50 MB | JPEG or TIFF |
| Gumroad | 816px | 1302px | ~2:3 | 5 MB | JPEG, PNG |
| Gumroad (recommended) | 1280px | 1920px | 2:3 | 5 MB | JPEG, PNG |
| Payhip | 2560px | 1600px | **8:5 LANDSCAPE** | 5 MB | JPEG, PNG |
| D2D / Writing Life | 1600px | 2400px | 2:3 (1:1.5) | 10 MB | JPEG or PNG |
| Apple Books (via D2D) | 1600px | 2400px | 2:3 | 10 MB | JPEG or PNG |
| Kobo (via D2D) | 1600px | 2400px | 2:3 | 10 MB | JPEG or PNG |
| Kobo (direct) | 1600px | 2400px | 2:3 | 5 MB | JPEG, PNG, GIF |
| Google Play Books | 1280px | 1920px | 2:3 | 2 MB | JPEG or PNG |
| Barnes & Noble (via D2D) | 1400px | 2100px | 2:3 | 5 MB | JPEG |

---

## Universal Production Sizes

To cover all platforms from a single master file, export these sizes:

### Master file (design at this resolution):
- **3200x4800px** at 300 DPI, sRGB color space
- Save as lossless PNG or TIFF for editing

### Export set:

| Export Name | Dimensions | Use For |
|---|---|---|
| cover-kdp.jpg | 1600x2560 | Amazon KDP |
| cover-gumroad.jpg | 1280x1920 | Gumroad |
| cover-payhip.jpg | 2560x1600 | Payhip (LANDSCAPE — rotate/crop) |
| cover-d2d.jpg | 1600x2400 | D2D, Apple Books, Kobo |
| cover-google.jpg | 1280x1920 | Google Play Books |
| cover-thumbnail.jpg | 500x750 | Website thumbnails, social sharing |

---

## Platform-Specific Notes

### Amazon KDP
- Ideal: 2560x1600 at 300 DPI (they recommend going larger)
- No text smaller than 7pt after scaling to thumbnail
- Title must be readable at thumbnail size (120x180px)
- No pricing, "free", or promotional text on cover
- No blurry or pixelated images
- Background cannot be white if it blends with the product page

### Gumroad
- Cover displays as card thumbnail on store page
- Square-ish crops may appear in Discover — ensure title is centered
- Animated GIFs supported for cover (unique to Gumroad)
- Recommended: add subtle branding/series indicator

### Payhip (LANDSCAPE)
- **This is the outlier** — 2560x1600 (landscape, not portrait)
- You cannot simply rotate the portrait cover
- Design a separate landscape version or use a wide crop with title overlay
- Displays prominently on the product page — high visual impact

### D2D / Writing Life
- Must be at least 1600px on the shortest side
- Cover will be distributed to all D2D retailers
- Retailers may crop or add borders — keep important elements away from edges
- Safe zone: keep text/key art within 90% of the center area

### Google Play Books
- Minimum 1280x1920, but 1600x2400 recommended
- Displayed smaller than most platforms — bold, simple designs work best
- No watermarks or "sample" text

---

## Color & Format Guidelines

| Property | Recommendation |
|---|---|
| Color space | sRGB (not CMYK — screens only) |
| DPI | 300 for master, 72-150 acceptable for final export |
| Format | JPEG at 85-95% quality for uploads (best size/quality ratio) |
| Background | Avoid pure white (#FFFFFF) — use off-white or add a subtle border |
| Text contrast | Ensure title readable on the actual background, not just in the editor |
| File naming | Lowercase, hyphens: `geo-pocket-guide-cover-kdp.jpg` |

## Cover Design Checklist

- [ ] Title readable at 120x180px thumbnail size
- [ ] Author name visible but not dominant
- [ ] Series branding consistent across all books in library
- [ ] No promotional text ("Free!", "Best Seller", prices)
- [ ] No blurry, stretched, or pixelated elements
- [ ] All text within safe zone (90% center area)
- [ ] Exported at correct dimensions for each platform
- [ ] Payhip landscape version prepared separately
- [ ] File size within platform limits
- [ ] sRGB color space (not CMYK)
