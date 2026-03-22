# Security Policy

## Scope

This Claude Code skill comprises markdown instruction files with no executable code,
authentication tokens, or user data. Security risks are limited to potentially harmful
commands in code snippets or misleading technical guidance that could expose servers
or API credentials.

## Reporting a Vulnerability

If you find content that could cause harm — incorrect commands that could damage a
server, misleading instructions that could expose API keys, or code snippets with
security flaws — please report it privately.

### How to report

1. **Preferred**: Use [GitHub Security Advisories](https://github.com/arturseo-geo/seo-geo-skill/security/advisories/new)
2. **Alternative**: DM via [𝕏 @TheGEO_Lab](https://x.com/TheGEO_Lab)

### What to include

- Which file and section contains the issue
- What the security risk is
- Suggested fix (if you have one)

### Response time

We will acknowledge reports within 48 hours and publish a fix within 7 days for
confirmed issues.

## Best Practices

When using this skill:
- Never commit `.env` files or API keys to version control
- Use the `.gitignore` provided (excludes `.env` and `node_modules/`)
- Store GSC credentials JSON outside the web-accessible directory
- Use basic auth on any public-facing SEO dashboards
- Add `X-Robots-Tag: noindex` headers to prevent indexing of internal tools
