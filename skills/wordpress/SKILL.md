---
name: wordpress
description: Publish, manage, and optimize WordPress content via the WordPress REST API. Use this skill whenever the user wants to create or update WordPress posts, pages, or custom post types; manage categories, tags, or taxonomies; upload media or set featured images; work with Gutenberg blocks; configure WooCommerce products; query or bulk-edit content; or automate any WordPress workflow. Trigger for any mention of WordPress, WP, WooCommerce, Gutenberg, wp-json, wp-cli, plugins, themes, or publishing to a WordPress site. Also trigger when the user wants to migrate, audit, or batch-process WordPress content.
---

# WordPress Skill

## Authentication Setup
WordPress REST API requires authentication. Ask the user for:
1. **Site URL** (e.g., `https://example.com`)
2. **Auth method** — prefer Application Passwords (WP 5.6+):
   - Users → Profile → Application Passwords → Add New
   - Returns `username:app-password` — base64 encode for Basic Auth header
3. **API base**: `{site_url}/wp-json/wp/v2/`

```javascript
const headers = {
  'Authorization': 'Basic ' + btoa('username:app-password'),
  'Content-Type': 'application/json'
};
```

---

## Core Operations

### Create / Update a Post
```bash
POST /wp-json/wp/v2/posts
{
  "title": "Post Title",
  "content": "<p>HTML or Gutenberg blocks</p>",
  "status": "draft" | "publish" | "private" | "future",
  "slug": "post-slug",
  "excerpt": "Short summary",
  "categories": [id1, id2],
  "tags": [id1, id2],
  "featured_media": media_id,
  "meta": { "custom_field": "value" }
}

# Update: PATCH /wp-json/wp/v2/posts/{id}
```

### Get Posts (with filtering)
```bash
GET /wp-json/wp/v2/posts?status=publish&per_page=100&page=1
GET /wp-json/wp/v2/posts?search=keyword&categories=5
GET /wp-json/wp/v2/posts?orderby=modified&order=desc
```

### Upload Media
```bash
POST /wp-json/wp/v2/media
Content-Disposition: attachment; filename="image.jpg"
Content-Type: image/jpeg
[binary data]
```

### Manage Taxonomies
```bash
GET  /wp-json/wp/v2/categories
POST /wp-json/wp/v2/categories  { "name": "New Cat", "slug": "new-cat", "parent": 0 }
GET  /wp-json/wp/v2/tags
POST /wp-json/wp/v2/tags  { "name": "New Tag", "slug": "new-tag" }
```

### Custom Post Types
```bash
# CPTs must have show_in_rest: true in registration
GET /wp-json/wp/v2/{cpt_slug}
POST /wp-json/wp/v2/{cpt_slug}
```

---

## Gutenberg Block Patterns
When writing content as Gutenberg blocks (use when site uses block editor):

```html
<!-- wp:paragraph -->
<p>Text here</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":2} -->
<h2>Heading</h2>
<!-- /wp:heading -->

<!-- wp:image {"id":123,"sizeSlug":"full"} -->
<figure class="wp-block-image"><img src="url" alt=""/></figure>
<!-- /wp:image -->

<!-- wp:list -->
<ul>
  <li>Item one</li>
  <li>Item two</li>
</ul>
<!-- /wp:list -->

<!-- wp:quote -->
<blockquote class="wp-block-quote"><p>Quote text</p></blockquote>
<!-- /wp:quote -->

<!-- wp:code -->
<pre class="wp-block-code"><code>code here</code></pre>
<!-- /wp:code -->
```

---

## WooCommerce Products
```bash
# Requires WooCommerce REST API (separate auth with consumer key/secret)
POST /wp-json/wc/v3/products
{
  "name": "Product Name",
  "type": "simple" | "variable" | "grouped",
  "regular_price": "29.99",
  "sale_price": "19.99",
  "description": "...",
  "short_description": "...",
  "categories": [{"id": 9}],
  "images": [{"src": "url", "alt": "alt text"}],
  "manage_stock": true,
  "stock_quantity": 100
}
```

---

## Batch Operations
For bulk tasks (update 50+ posts, migrate content), generate a Python or Node.js script:
1. Authenticate once, store token
2. Paginate with `per_page=100&page=N`
3. Process with rate limiting (sleep 0.5s between writes)
4. Log successes and failures to CSV

---

## Common Workflows

### Publish a new blog post end-to-end:
1. Check/create categories → get IDs
2. Upload featured image → get media ID
3. Write/format content as HTML or Gutenberg blocks
4. POST to /posts with all fields
5. Return the published URL

### Audit existing content:
1. GET all posts (paginate)
2. Check: missing excerpts, missing featured images, draft posts older than 90 days, duplicate slugs
3. Return prioritized fix list

### SEO metadata update (Yoast/RankMath via custom fields):
```json
"meta": {
  "_yoast_wpseo_title": "SEO Title",
  "_yoast_wpseo_metadesc": "Meta description",
  "_yoast_wpseo_focuskw": "focus keyword"
}
```

---

## Error Handling
| Code | Meaning | Fix |
|---|---|---|
| 401 | Bad credentials | Check app password, re-encode base64 |
| 403 | Insufficient permissions | User needs Editor/Admin role |
| 404 | Post/endpoint not found | Check CPT has `show_in_rest: true` |
| 422 | Validation error | Check required fields and data types |

Always return the post ID and permalink on success.
