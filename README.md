<img width="1850" height="733" alt="image" src="https://github.com/user-attachments/assets/94b67421-99c8-4f6c-99e8-8b60ec1add2c" />

# 🎯 Job Detector — Automated LinkedIn Job Search with AI-Powered CV Generation

An automated n8n workflow that searches, analyzes, and scores LinkedIn vacancies, sends personalized notifications to Telegram, and generates tailored PDF resumes for each high-scoring job — all with an interactive Telegram bot for on-the-fly CV customization.

## Table of Contents

- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Features](#features)
- [Installation](#installation)
- [Credentials Setup](#credentials-setup)
- [Configuration](#configuration)
- [CV Themes](#cv-themes)
- [Telegram Bot Commands](#telegram-bot-commands)
- [Cost Breakdown](#cost-breakdown)
- [Limitations](#limitations)
- [Strengths](#strengths)

## How It Works

The workflow runs in three parallel modes:

**Mode 1 — Scheduled Scan (every 4 hours):** Scrapes LinkedIn for jobs matching configured keywords and location, filters duplicates, scores each vacancy via LLM, analyzes resume fit, generates a tailored PDF resume, and sends everything to Telegram.

**Mode 2 — Gmail Trigger (every 30 minutes):** Polls for unread LinkedIn Job Alert emails, extracts job URLs, scrapes job details via a separate Apify actor, normalizes the data into a unified format, and feeds it into the same processing pipeline.

**Mode 3 — Telegram Bot (real-time):** Listens for callback actions from Telegram inline buttons. Allows the user to regenerate a CV with a custom prompt, change the CV theme, or adjust the PDF scale — all from within the Telegram chat.

All three modes converge at a centralized **Configuration** node that holds every tunable parameter: CV data, search criteria, thresholds, templates, and PDF options.

## Architecture

```mermaid
flowchart TD
    subgraph SOURCES
        A["⏰ Cron (every 4h)"]
        B["📧 Gmail (30min poll)"]
        C["🤖 TG Bot Trigger"]
    end

    A --> SA["Set source=cron"]
    B --> SB["Set source=gmail"]
    C --> SC["Set source=tg"]

    SA & SB & SC --> CFG["⚙️ Configuration\n(CV, thresholds, search params)"]
    CFG --> VALID{"Config valid?"}
    VALID -->|No| STOP["⛔ Stop"]
    VALID -->|Yes| ROUTER{"General Router"}

    ROUTER -->|source=cron| FIND["🔍 Find LinkedIn Jobs\n(Apify keyword search)"]
    ROUTER -->|source=gmail| PARSE["📝 Parse Job URLs\nfrom email"]
    ROUTER -->|source=tg| REGEN["🔄 Parse Regen Action"]

    PARSE --> ALERT["🔍 Find Alerted LinkedIn Jobs\n(Apify URL scraper)"]
    ALERT --> NORM["🔄 Normalize Fields"]

    FIND & NORM --> DEDUP["🔒 Remove Duplicates (jobId)"]
    DEDUP --> FILTER["🗑️ Filter expired/closed/broken"]

    REGEN --> REGROUTER{"Regen Router"}
    REGROUTER -->|Theme| THEME["🎨 Change Theme"]
    REGROUTER -->|Scale| SCALE["📐 Change Scale"]
    REGROUTER -->|Prompt| PROMPT["✏️ Regenerate with Prompt"]

    subgraph JOB_PROCESSING ["JOB PROCESSING"]
        FILTER --> LOOP["🔁 Iterate vacancy (batch)"]
        LOOP --> DELAY["⏳ 5s delay"]
        DELAY --> KWCHECK{"Keyword match\n≥ threshold?"}
        KWCHECK -->|No| LOOP
        KWCHECK -->|Yes| LLM1["🤖 LLM: calculate job relevance\n(score 0-100, cover letter, TG message)"]
        LLM1 --> GATE{"Score ≥ threshold?"}
        GATE -->|No| LOOP
        GATE -->|Yes| CVSUG["📊 Find CV suggestions (LLM)"]
        CVSUG --> IMPROVE["🤖 LLM: improve resume fields"]
        IMPROVE --> NORMMSG["🤖 LLM: normalize message → Ukrainian"]
        NORMMSG --> BUILD["🏗️ Build themed HTML CV"]
        BUILD --> PDF["📄 Convert to PDF (pdfspark.dev)"]
        PDF --> UPSERT["💾 Upsert CV to Data Table"]
    end

    subgraph DELIVERY
        UPSERT --> TG_MSG["📱 Telegram: job summary (HTML)"]
        TG_MSG --> TG_DOC["📄 Telegram: tailored PDF resume"]
        TG_DOC --> BUTTONS["Inline buttons:\n✏️ Перегенерувати | 📐 Масштаб\n🎨 Тема | 🔄 Тема за замовч."]
        BUTTONS --> DONE["✅ Finished → next vacancy"]
        DONE --> LOOP
    end
```

## Features

**Search & Collection:**
- Automated search across customizable keywords (configured in the Configuration node)
- Gmail polling for instant processing of LinkedIn Job Alert emails (every 30 min)
- Deduplication of jobs between runs (remembers processed jobIds)
- Deduplication of email messages (marks as read after processing)
- Keyword matching — automatic comparison of job stack vs your resume keywords
- Filters out expired, closed, offline, and paused job listings
- Filters out broken listings with no description

**AI Analysis:**
- Job relevance scoring from 0 to 100 via LLM (Gemini Flash)
- Personalized cover letter generation in Ukrainian (for score ≥ 85)
- Resume gap analysis — skill gaps, ATS compatibility, improvement actions
- LLM-driven resume improvement — rephrases experience bullets to include ATS keywords
- Translation and normalization of all messages to Ukrainian

**CV Generation:**
- Automatic tailored PDF resume for each qualifying job
- 11 themed HTML→PDF templates (Classic, Anthropic, Gemini, ChatGPT, DeepSeek, Grok, Perplexity, VSCode, React, Behance, Figma)
- Locked/modifiable field split — name, contact, education stay fixed; headline, summary, experience, skills are optimized per job
- CV storage in n8n Data Table for later regeneration
- PDF conversion via pdfspark.dev API with configurable A4 options

**Telegram Bot:**
- HTML-formatted job notifications with clickable links
- PDF resume attached to each notification
- Inline buttons: Regenerate (with custom prompt), Scale, Theme, Default Theme
- Interactive conversation flow: the bot asks for input, waits, and applies changes
- Regenerated CVs are re-sent as new PDF documents

**Filtering:**
- Pre-filter by keyword match (configurable threshold, default < 33% → skipped without LLM call, saves tokens)
- Score-based filter (configurable threshold, default < 85 → no CV analysis or notification)
- Status-based filter (expired/closed/offline/paused → skipped)

## Installation

**Prerequisites:**
- n8n instance (self-hosted or n8n cloud)
- Accounts: Apify, Google Cloud (Gemini API), Telegram Bot
- Gmail account with LinkedIn Job Alerts configured
- An n8n Data Table named `cv_store` with columns: `label`, `htmlContent`, `resumeJson`, `company`, `theme`

**Import the workflow:**

1. Download `Job_Detector__23_.json` from this repository
2. In n8n, go to **Settings → Import from File**
3. Select the JSON file
4. The workflow will appear as "Job Detector"

Or via CLI:
```bash
n8n import:workflow --input=job_detector.json
```

**Post-import steps:**

1. Open the workflow and configure credentials for all nodes with an orange indicator
2. Open the **Configuration** node and replace all personal data (CV, chat ID, search criteria)
3. Create the `cv_store` Data Table in your n8n project and link it in the **Upsert CV's content** node

## Credentials Setup

After import, configure credentials for each service. Open the workflow and check nodes with the orange missing-credentials indicator.

### 1. Apify OAuth2 API

Used in 2 nodes: **Find LinkedIn Jobs**, **Find Alerted LinkedIn Jobs**.

1. Register at [apify.com](https://apify.com)
2. Go to **Settings → Integrations → API tokens**
3. Create a new token
4. In n8n: **Credentials → New → Apify OAuth2 API** → paste the token

**Apify actors used:**
- `2rJKkhh7vjpX7pvjg` — LinkedIn Jobs Scraper (keyword-based search)
- `HxVzxLvPwxYkc6ndm` — LinkedIn Jobs Scraper - Professional Job Listings (URL-based detail scraping)

### 2. Google Gemini (LLM)

Used in 4+ Gemini model nodes throughout the workflow.

1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Create an API key
3. In n8n: **Credentials → New → Google Gemini API** → paste the key
4. Default model: `gemini-3.1-flash-lite` (fast and cheap, but you can choose another)

### 3. Telegram Bot

Used in multiple nodes: **Send a text message**, **Send a document to TG**, **TG: Regen Trigger**, and all interactive prompt/response nodes.

1. Message [@BotFather](https://t.me/BotFather) in Telegram
2. Create a new bot: `/newbot`
3. Copy the bot token
4. Find your Chat ID — message [@userinfobot](https://t.me/userinfobot)
5. In n8n: **Credentials → New → Telegram API** → paste the token
6. In the **Configuration** node, set `tg_chat_id` to your Chat ID

### 4. Gmail OAuth2

Used in nodes: **Get many messages**, **Mark a message as read**.

1. Create OAuth2 credentials in [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Enable the Gmail API
3. In n8n: **Credentials → New → Gmail OAuth2 API** → complete the OAuth flow
4. Make sure you have LinkedIn Job Alerts enabled and arriving in your inbox

### 5. PDF Service

The workflow uses `https://pdfspark.dev/api/v1/pdf/from-html` for HTML→PDF conversion. This is called via HTTP Request nodes (**Make a PDF**, **Rescale: Make PDF**). Check if pdfspark.dev requires an API key and configure it in the HTTP Request node headers if needed.

## Configuration

All settings live in the **Configuration** node — a single Set node with a JSON object. Open it and customize:

### Telegram Chat ID

```json
"tg_chat_id": 123456789
```
Replace with your own Chat ID.

### Score Threshold

```json
"score_threshold": 85
```
Minimum LLM relevance score for a job to trigger CV analysis and notification. Jobs scoring below this are silently skipped.

### Keyword Match Threshold

```json
"keyword_match_threshold": 33
```
Minimum keyword match percentage for a job to be sent to the LLM at all. Jobs below this threshold are skipped without consuming LLM tokens.

### Search Keywords

```json
"linkedin_job_search_criterias": {
  "keyword": [
    "React Native",
    "Full Stack",
    "Node.js",
    "JavaScript",
    "TypeScript",
    "Front End"
  ],
  "location": "Kyiv, Ukraine",
  "maxItems": 150,
  "publishedAt": "r86400",
  "resumeKeywords": [
    { "keyword": "JavaScript", "aliases": ["JS"] },
    { "keyword": "TypeScript", "aliases": ["TS"] },
    { "keyword": "Node.js", "aliases": ["Node", "NodeJS"] },
    { "keyword": "React Native", "aliases": ["RN"] },
    { "keyword": "React" },
    { "keyword": "Expo" }
  ],
  "saveOnlyUniqueItems": true
}
```

- `keyword` — search terms sent to LinkedIn
- `location` — target job location
- `maxItems` — maximum results per scraper run
- `publishedAt` — time window (`r86400` = last 24 hours, `r43200` = last 12 hours)
- `resumeKeywords` — your skill keywords for automatic stack matching; `aliases` let you catch variations

### Candidate CV

```json
"candidate_raw_cv": {
  "full_name": "YOUR NAME",
  "headline": "Your headline",
  "contact": { "phone": "...", "email": "...", "linkedin": "...", "location": "..." },
  "summary": "Your summary...",
  "experience": [ ... ],
  "education": [ ... ],
  "languages": [ ... ],
  "achievements": [ ... ],
  "skills": [ ... ]
}
```
Replace the entire object with your own CV data. This is used for LLM scoring, cover letter generation, resume improvement, and PDF generation. The structure must be preserved — the workflow splits it into locked fields (`full_name`, `contact`, `education`, `languages`, `achievements`) and modifiable fields (`headline`, `summary`, `experience`, `skills`).

### Candidate Experience Level

```json
"candidate_experience_level": "mid"
```
Used by the LLM to calibrate scoring. Options: `junior`, `mid`, `senior`.

### CV Theme

```json
"cv_theme": "chatgpt"
```
Default theme for generated resumes. See [CV Themes](#cv-themes) for all options.

### PDF Options

```json
"pdf_options": {
  "format": "A4",
  "printBackground": true,
  "emulateMediaType": "screen",
  "margin": { "top": "0mm", "right": "0mm", "bottom": "0mm", "left": "0mm" }
}
```

> **Tip:** Change `"emulateMediaType"` to `"print"` to make templates more printer-friendly. The output will look a bit duller (muted colors, simplified backgrounds) but is better suited for physical printing.

### Message Template

The `message_template` field contains the HTML template for Telegram notifications. Placeholders like `[Job Title]`, `[Company Name]`, `[Score]` are replaced by the LLM normalization step.

## CV Themes

The workflow includes 11 visual themes for PDF resumes, each implemented as a standalone HTML builder node:

| Theme | Style |
|-------|-------|
| `classic` | Clean, traditional resume layout |
| `anthropic` | Inspired by Anthropic's brand aesthetic |
| `gemini` | Google Gemini-inspired design |
| `chatgpt` | OpenAI ChatGPT-inspired design |
| `deepseek` | DeepSeek-inspired design |
| `grok` | xAI Grok-inspired design |
| `perplexity` | Perplexity AI-inspired design |
| `vscode` | VS Code editor-inspired light theme |
| `react` | React.js documentation-inspired design |
| `behance` | Behance portfolio-inspired creative layout |
| `figma` | Figma-inspired design tool aesthetic |

Set the default theme in the Configuration node (`cv_theme` field), or change it per-CV via the Telegram bot's theme picker.

## Telegram Bot Commands

After receiving a job notification with an attached PDF resume, the user sees inline buttons:

| Button | Action |
|--------|--------|
| ✏️ Перегенерувати | Enter a custom prompt (e.g., "emphasize backend experience") and the LLM regenerates the resume accordingly |
| 📐 Масштаб | Adjust the PDF scale factor (useful if content overflows one page) |
| 🎨 Тема | Switch to a different CV theme for this specific resume |
| 🔄 Тема за замовч. | Change the default theme used for all future resumes |

The bot uses n8n's **Wait** nodes to pause execution and wait for user input, creating an interactive conversational flow entirely within Telegram.

## Cost Breakdown

| Service | Cost | Notes |
|---------|------|-------|
| LinkedIn Jobs Scraper (keyword search) | ~$0.60 / 1,000 jobs | ~20-30 jobs per query |
| LinkedIn Jobs Scraper (URL detail) | ~$2 / 1,000 jobs | Limited to 1 job per request |
| Google Gemini Flash Lite | Free tier | Multiple LLM calls per qualifying job |
| pdfspark.dev | Check pricing | HTML→PDF conversion |
| Telegram | Free | Bot API |
| Gmail | Free | OAuth2 API |

At 6 runs per day (every 4 hours) with the Gmail trigger running every 30 minutes: approximately **~$0.10-0.15/day** or **~$3-5/month** on Apify.

To reduce costs: use fewer search keywords, increase the cron interval, lower `maxItems`, use `publishedAt: "r43200"` (12 hours instead of 24), or raise the `keyword_match_threshold` to send fewer jobs to LLM.

## Limitations

- **Single user.** The workflow is designed for one candidate. The Configuration node holds one CV and one chat ID.
- **LinkedIn scraping dependency.** Relies on Apify actors which may break if LinkedIn changes its page structure.
- **pdfspark.dev dependency.** PDF generation depends on an external API. If it goes down, CVs won't be generated.
- **No persistent job deduplication across workflow versions.** Deduplication relies on n8n's built-in Remove Duplicates node which tracks within execution scope. If you reimport the workflow, previously seen jobs may reappear.
- **Gmail polling, not push.** The Gmail source polls every 30 minutes, so there can be up to a 30-minute delay for email-triggered jobs.
- **Rate limiting is basic.** A 5-second delay between job iterations. Heavy runs with many qualifying jobs may take significant time.
- **LLM hallucination risk.** Cover letters and resume improvements are generated by LLM. Review before sending to employers.
- **Data Table coupling.** The CV regeneration flow requires an n8n Data Table (`cv_store`) to be created and linked manually.

## Strengths

- **End-to-end automation.** From job discovery to tailored resume delivery — fully hands-off for the daily flow.
- **Centralized configuration.** Everything in one node — easy to understand, modify, and maintain.
- **Smart cost optimization.** Two-stage filtering (keyword match → LLM score) avoids wasting tokens on irrelevant jobs.
- **Tailored resumes at scale.** Each qualifying job gets a purpose-built CV with ATS-optimized keywords, not a generic one.
- **Interactive Telegram bot.** Regenerate, restyle, and rescale CVs without opening n8n or any other tool.
- **11 visual themes.** Professional variety for different industries and personal taste.
- **Dual input sources.** Cron-based search catches broad results; Gmail alerts catch jobs matching LinkedIn's own recommendation engine.
- **Ukrainian localization.** All messages, cover letters, and notifications are automatically translated to Ukrainian.
- **Validation gate.** The workflow validates configuration on every run and stops early if required fields are missing.
- **CV versioning via Data Table.** Previously generated resumes are stored and can be regenerated or restyled without re-running the full pipeline.

<img width="1684" height="777" alt="image" src="https://github.com/user-attachments/assets/5a40f3c1-fd32-47e0-b4a8-859c4d3115c4" />

# 📰 News Feed Automation

An automated n8n workflow that collects articles from RSS feeds every morning, generates short Ukrainian-language summaries using Google Gemini, and publishes them to a Telegram channel with images.

---

## Architecture

```mermaid
flowchart TD
    SCHED["⏰ Schedule Trigger\n(10:00 AM)"] --> URLS["📋 URL List\n(9 feeds)"]
    URLS --> SPLIT["Split Out"]
    SPLIT --> RSS["📡 RSS Read"]
    RSS --> FRESH["🔍 Filter fresh\n(current year only)"]
    FRESH --> RAND["🔀 Randomize"]
    RAND --> LIM["✂️ Limit to 15"]
    LIM --> DEDUP["🔒 Remove Duplicates\n(cross-execution)"]
    DEDUP --> LOOP["🔁 Loop Over Items\n(batch by 1)"]
    LOOP --> WAIT["⏳ Wait 5s\n(rate limiting)"]
    WAIT --> LLM["🤖 Gemini 2.5 Flash\n(summarize)"]
    LLM --> IMG["🖼️ Map Images\n(fallbacks)"]
    IMG --> TG["📱 Send Photo\nto Telegram"]
    TG --> LOOP
```

---

## What the Workflow Does

1. **Scheduled trigger** — runs daily at 10:00 AM.
2. **RSS collection** — iterates over 9 RSS feeds (tech, AI, Ukrainian media).
3. **Filtering** — keeps only articles from the current year.
4. **Randomization + limit** — shuffles and trims to 15 articles.
5. **Deduplication** — removes articles already sent in previous runs (keyed by `creator + link`).
6. **LLM summary** — Gemini 2.5 Flash generates a short annotation in Ukrainian (up to 1024 characters, Telegram HTML).
7. **Image selection** — takes `enclosure.url` from the article or falls back to a domain-specific logo.
8. **Publishing** — sends a photo with caption to the Telegram chat.
9. **Rate limiting** — 5-second pause between iterations to avoid hitting API limits.

---

## RSS Sources

| Source | URL |
|--------|-----|
| Hacker News (frontpage) | `https://hnrss.org/frontpage` |
| DEV.to | `https://dev.to/feed/` |
| OpenAI Blog | `https://openai.com/news/rss.xml` |
| Google DeepMind Blog | `https://deepmind.google/blog/rss.xml` |
| CSS-Tricks | `https://css-tricks.com/feed/` |
| AIN.UA | `https://ain.ua/feed/` |
| ITPro | `https://www.itpro.com/feeds.xml` |
| ITC.UA | `https://itc.ua/ua/feed/` |
| UNIAN | `https://rss.unian.net/site/news_ukr.rss` |

---

## Requirements

- **n8n** — self-hosted or n8n Cloud
- **Google Gemini API** — API key for the `gemini-2.5-flash` model
- **Telegram Bot** — bot token + chat ID for publishing

---

## Setup

### 1. Import the Workflow

1. Open n8n → **Workflows** → **Import from File**.
2. Upload the workflow JSON file.

### 2. Credentials

You need to create two credentials in n8n:

**Google Gemini:**
- Go to **Settings** → **Credentials** → **New Credential**.
- Type: `Google Gemini Chat Model`.
- Paste your Google AI Studio API key.

**Telegram Bot:**
- Type: `Telegram API`.
- Paste the bot token (get it via [@BotFather](https://t.me/BotFather)).

### 3. Telegram Chat Configuration

In the **"Send a post to TG"** node, replace `chatId` with your channel or chat ID.

### 4. Customizing Sources

Edit the URL array in the **"Url sources"** node. Add or remove RSS feeds as needed.

### 5. Activation

Toggle the workflow on — it will run automatically every day at 10:00 AM.

---

## Node Structure

| Node | Type | Purpose |
|------|------|---------|
| Call every morning (10AM) | Schedule Trigger | Daily trigger |
| Url sources | Set | Array of RSS feed URLs |
| Split Out | Split Out | Expands array into individual items |
| RSS Read | RSS Feed Read | Reads articles from each RSS feed |
| Pick fresh posts | Filter | Filters by current year |
| Randomize | Sort (random) | Shuffles articles |
| Limit to 15 | Limit | Caps the number of articles |
| Remove Duplicates | Remove Duplicates | Excludes previously sent articles |
| Loop Over Items | Split In Batches | Processes one article at a time |
| If not the first iteration | If | Skips the pause for the first item |
| Wait 5 seconds | Wait | Pause between publications |
| LLM: process a post | Chain LLM | Summary generation (Gemini) |
| Google Gemini Chat Model | LM Chat | Model for the LLM node |
| Map images | Code | Image selection or fallback |
| Send a post to TG | Telegram | Publishes to Telegram |

---

## Telegram Message Format

Each post is sent as a photo with an HTML caption:

```
<b>Article Title</b>

<i>Author:</i> Name
<i>Date:</i> 17.05.2026
<i>Category:</i> AI, Machine Learning

Short annotation in Ukrainian — 2-3 sentences
about the article and why it might be useful.

<a href="https://...">Read full article</a>
```

---

## Fallback Images

If an article has no `enclosure.url`, a source logo is used instead:

| Domain | Fallback |
|--------|----------|
| openai.com | OpenAI logo |
| deepmind.google | DeepMind logo |
| ain.ua | AIN logo |
| itc.ua | ITC logo |
| dev.to | DEV logo |
| css-tricks.com | CSS-Tricks logo |
| itpro.com | ITPro logo |
| unian.net | UNIAN logo |
| (other) | Stock news image |

---

## Possible Issues

- **RSS unavailable** — the RSS Read node has `onError: continueRegularOutput`, so a single feed failure won't stop the entire workflow.
- **Gemini rate limit** — the 5-second pause between iterations minimizes risk, but increase the delay if you process more articles.
- **Telegram rate limit** — Telegram limits bots to ~30 messages per second; 15 messages with pauses shouldn't cause issues.
- **Fallback image unavailable** — replace the URL in the "Map images" node with an alternative.

---

<img width="1684" height="778" alt="image" src="https://github.com/user-attachments/assets/a5834121-9ed7-465e-be1f-bc896c47a50c" />

---

# 🛒 AliExpress Scraper via Apify

An automated n8n workflow that scrapes products from AliExpress weekly via Apify, normalizes the data, and publishes product cards to Telegram with prices, discounts, and ratings.

---

## Architecture

```mermaid
flowchart TD
    SCHED["⏰ Schedule Trigger\n(Sat 12:00)"] --> APIFY["🕷️ Apify Actor\n(AliExpress Scraper)"]
    APIFY -->|success| DEDUP["🔒 Remove Duplicates\n(by productId)"]
    APIFY -->|error| ERR["⚠️ Send error alert\nto Telegram"]
    DEDUP --> NORM["⚙️ Normalize Data\n(Code node)"]
    NORM --> TG["📱 Send photo message\nto Telegram"]
```

---

## What the Workflow Does

1. **Scheduled trigger** — runs every Saturday at 12:00.
2. **Scraping** — Apify actor `mfblss3fLQRaqhg6K` searches products by queries (`fpv`, `graphic cards`, `LiPo`, `vtx`), sorted by rating, up to 10 products per query.
3. **Error handling** — if the actor fails after 3 retries, a Telegram alert is sent with error details.
4. **Deduplication** — removes duplicates by `productId`.
5. **Normalization** — a Code node transforms raw Apify data into a clean format: price, discount, rating, image, coupons, etc.
6. **Publishing** — each product is sent to Telegram as a photo with an HTML caption.

---

## Search Queries

By default, the workflow searches for:

| Query | Description |
|-------|-------------|
| `fpv` | FPV drones and components |
| `graphic cards` | Graphics cards |
| `LiPo` | LiPo batteries |
| `vtx` | Video transmitters for FPV |

To change the queries, edit the `queries` array in the **"Run an Actor and get dataset"** node.

---

## Requirements

- **n8n** — self-hosted or n8n Cloud
- **Apify** — account + OAuth2 credentials (or API token)
- **Telegram Bot** — bot token + chat ID

---

## Setup

### 1. Import the Workflow

1. Open n8n → **Workflows** → **Import from File**.
2. Upload the workflow JSON file.

### 2. Credentials

**Apify:**
- Go to **Settings** → **Credentials** → **New Credential**.
- Type: `Apify OAuth2 API`.
- Authorize via Apify or paste your API token from [Apify Console](https://console.apify.com/account#/integrations).

**Telegram Bot:**
- Type: `Telegram API`.
- Paste the bot token (get it via [@BotFather](https://t.me/BotFather)).

### 3. Telegram Chat ID Configuration

Fill in the `chatId` in two nodes:

- **"Send a photo message"** — main product publications.
- **"Send an error message"** — error alerts.

To find your chat ID:
- Send a message to your bot.
- Make a request: `https://api.telegram.org/bot<TOKEN>/getUpdates`.
- Find `chat.id` in the response.

### 4. Search Configuration

In the **"Run an Actor and get dataset"** node, edit `customBody`:

```json
{
  "deduplicateProducts": true,
  "includeDescription": true,
  "includeQuestions": false,
  "includeReviews": false,
  "includeShippingDetails": false,
  "includeVariants": false,
  "maxItems": 10,
  "queries": ["fpv", "graphic cards", "LiPo", "vtx"],
  "sortBy": "rating"
}
```

| Parameter | Description |
|-----------|-------------|
| `maxItems` | Max products per query |
| `queries` | Array of search queries |
| `sortBy` | Sort order: `rating`, `price`, `orders` |
| `includeDescription` | Include product description |
| `includeReviews` | Include reviews (increases run time) |

### 5. Activation

Toggle the workflow on — it will run automatically every Saturday at 12:00.

---

## Node Structure

| Node | Type | Purpose |
|------|------|---------|
| Schedule Trigger | Schedule Trigger | Weekly trigger (Saturday, 12:00) |
| Run an Actor and get dataset | Apify | Runs the AliExpress scraper |
| Remove Duplicates | Remove Duplicates | Deduplication by `productId` |
| Normalize Data | Code | Transforms data into a clean format |
| Send a photo message | Telegram | Publishes a product card |
| Send an error message | Telegram | Error alert |

---

## Telegram Message Format

Each product is sent as a photo with a caption:

```
Product Title

💰 $12.99  $29.99  (-57%)
⭐ 4.8  •  📦 1,234 sold
🏷️ $2 off every $20

🔗 View on AliExpress
```

---

## Normalized Fields

The `Normalize Data` Code node extracts the following fields:

| Field | Source | Fallback |
|-------|--------|----------|
| `productId` | `d.productId` | `''` |
| `title` | `d.title.displayTitle` | `''` |
| `currentPrice` | `d.prices.salePrice.minPrice` | `d.extractedData.currentPrice` |
| `originalPrice` | `d.prices.originalPrice.minPrice` | `d.extractedData.originalPrice` |
| `discount` | `d.extractedData.discountPercent` | `d.prices.salePrice.discount` |
| `rating` | `d.evaluation.starRating` | `d.extractedData.rating` |
| `totalOrders` | `d.extractedData.totalOrders` | `0` |
| `imageUrl` | `d.image.imgUrl` | `d.images[0].imgUrl` |
| `storeName` | `d.store.storeName` | `''` |
| `coupons` | `d.sellingPoints[].tagContent.tagText` | `''` |

> If Apify changes the response schema, update the mapping in this node.

---

## Error Handling

- **Retry policy** — the Apify node automatically retries up to 3 times with a 2-second delay.
- **Error output** — on final failure, data goes to the second output → Telegram alert with error text and a link to Apify Console.
- **Continue on error** — `onError: continueErrorOutput` allows error handling instead of stopping the workflow.

---

## Possible Issues

- **Apify credits** — scraping consumes Apify credits; monitor your balance.
- **Actor schema changes** — if the actor author updates the format, fix the mapping in `Normalize Data`.
- **AliExpress blocking** — the actor may return fewer results with aggressive scraping; reduce `maxItems` or increase the interval.
- **Empty `chatId`** — don't forget to fill in the chat ID in both Telegram nodes.

---

## License

MIT — use, modify, and share freely.
