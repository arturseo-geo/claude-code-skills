# Research Tools & Utilities

## Web Discovery & Scraping

### Firecrawl

**What it does:** Web scraping, site mapping, LLM extraction

**Best for:** Discovering competitor websites, extracting structured data, crawling documentation

**Usage:**
```bash
firecrawl_scrape --url "https://competitor.com" --format json
firecrawl_map --url "https://competitor.com" --search "pricing features"
```

**Strength:** Handles JavaScript-rendered sites, extracts clean markdown

**Limitation:** Rate limited, costs tokens

### Puppeteer

**What it does:** Headless browser automation

**Best for:** JavaScript-heavy websites, clicking through paywalls, taking screenshots

**Usage:**
```bash
puppeteer_navigate --url "https://site.com"
puppeteer_screenshot --selector ".pricing-table"
```

**Strength:** Full browser control, JavaScript execution

**Limitation:** Slower than HTTP scraping

## People & Company Intelligence

### LinkedIn MCP

**What it does:** Search people, get company profiles, find job openings

**Best for:** Competitor team intelligence, founder research, skill/role mapping

**Usage:**
```bash
linkedin_search_people --keywords "VP Sales at Segment"
linkedin_get_company_profile --company_name "stripe" --sections "jobs,posts"
```

**Strength:** Real-time data, verified identity information

**Limitation:** Requires active Chromium session

## Data Analysis

### PostgreSQL

**What it does:** Query internal research databases

**Best for:** Consolidating research data, cross-referencing multiple sources

**Usage:**
```sql
SELECT competitor, market_share, growth_rate, founded_year
FROM market_research
WHERE market = 'SaaS'
ORDER BY market_share DESC;
```

### Google Sheets / CSV

**What it does:** Organize research into comparable matrices

**Best for:** Creating competitor matrices, TAM sizing, segment analysis

## Trend Detection

### Google Trends

**URL:** trends.google.com

**What it does:** Search volume trends, related searches, geographic interest

**Best for:** Trend validation, relative popularity comparison

**Usage:**
- Type "AI SEO" → See 5-year trend
- Compare "AI SEO" vs "traditional SEO" → See relative interest
- Filter by region/time period

### Patent Search (USPTO)

**URL:** patents.google.com

**What it does:** Track innovation trends by patent filings

**Best for:** Technology trend validation, R&D pipeline analysis

**Usage:**
- Search "AI large language model" → See filing trends
- Filter by assignee (company) → See R&D investment
- Filter by filing date → New technologies

### Job Posting Analysis

**Tools:** LinkedIn Jobs, Glassdoor, Angel List

**What it does:** Indicator of hiring trends, skill demand

**Best for:** Market timing, emerging role demand, competitive hiring

**Usage:**
- Search "AI engineer" → See volume over time
- Filter by company → See competitor hiring spree
- Filter by location → Geographic expansion signal

## Source Validation

### Fact-Checking

**Sites:** Snopes.com, FactCheck.org, PolitiFact.com

**What it does:** Validate extraordinary claims

**Best for:** Dispelling misinformation, validating viral claims

### Bias Detection

**Media Bias Chart:** mediabiaschartdotcom

**What it does:** Assess source credibility and political slant

**Best for:** Evaluating news source reliability

## Research Organization

### Notion

**What it does:** Create research databases, link sources, tag findings

**Best for:** Organizing research over time, building persistent knowledge base

**Usage:**
- Create database of competitors
- Tag by category, market, growth rate
- Link to source URLs
- Query across projects

### Obsidian

**What it does:** Local markdown-based knowledge base

**Best for:** Building interconnected research notes
**Usage:**
- Create note for each competitor
- Create note for each trend
- Link notes together
- Query backlinks to find patterns

## Comparison Matrix

| Tool | Best For | Cost | Speed |
|------|----------|------|-------|
| Firecrawl | Competitor website scraping | $$ | Fast |
| LinkedIn | People/company intelligence | $$ | Real-time |
| Statista | Market sizing | $$$ | Instant |
| IBISWorld | Industry reports | $$$ | Instant |
| Google Trends | Trend validation | Free | Instant |
| USPTO Patents | Technology trends | Free | Instant |
| Crunchbase | Startup funding data | $$ | Instant |
| Notion | Research organization | $ | Instant |
