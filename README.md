
# Schedule Monitor with API & Telegram Notifications

> **Evolution from [sitemonitor-screen](https://github.com/reginanka/sitemonitor-screen)**: This project migrates from screenshot-based visual monitoring to API-driven data monitoring while preserving the proven architecture of automated change detection and Telegram notifications.

## Overview

This monitor tracks changes in schedule/queue data by comparing hash values for each queue fetched from an API endpoint. When changes are detected for any queue, it sends a Telegram notification indicating which queue changed, along with a screenshot from the website for visual context.

**Key features:**
- API-first approach with per-queue hash comparison
- Detects *where* changes occurred (which queue), not *what* changed
- Screenshot attachment from website for user verification
- Automated monitoring via GitHub Actions (scheduled runs)
- Separate logging channel for observability
- Modular architecture for easy extension

## Why API-based monitoring?

### Previous approach (`sitemonitor-screen`)
- **How it worked**: Playwright rendered the page, captured a screenshot between two text anchors (specific HTML elements), and compared MD5 hashes
- **What it did well**: Isolated the exact content block between anchors, avoiding noise from ads, navigation, or other page elements
- **Limitation**: Monitored one section as a whole; to identify which specific queue changed would require screenshot analysis (OCR/image processing), which is slower and more complex than API comparison

### Current approach
- **Opportunity**: API endpoint became available, returning structured data for each queue (1.1, 1.2, 2.1, 2.2, 3.1, etc.)
- **Benefits**: 
  - Separate hash for each queue → pinpoint exactly which queue changed instantly
  - Faster: no full page render for initial detection, no OCR/image processing needed
  - Still captures screenshot from website for visual confirmation
  - User decides what changed by viewing screenshot or clicking link
- **Result**: Faster detection with precise queue identification, while keeping visual verification option

## How it works

┌─────────────────┐
│ GitHub Actions │ (scheduled trigger)
└────────┬────────┘
│
▼
┌──────────────────────────────────────────────┐
│ monitor.py (orchestrator) │
│ 1. Fetch API data for all queues │
│ 2. Compute hash for each queue │
│ 3. Compare with previous hashes │
│ 4. If hash differs for queue X: │
│ - Take screenshot from website │
│ - Send notification: "Queue X changed" │
│ - Attach screenshot + link │
│ 5. Buffer logs → send to log channel │
└──────────────────────────────────────────────┘
│
▼
┌─────────────────┐
│ data/ │
│ - current.json │ (latest API response for all queues)
│ - last_hash.json│ (hash per queue: {"1.1": "abc123", "2.1": "def456", ...})
└─────────────────┘

text

### Detailed flow
1. **Data acquisition**: `site_content.py` calls the API and returns data for all queues (1.1, 1.2, 2.1, etc.)
2. **Per-queue hashing**: `monitor.py` computes a hash for each queue's data individually
3. **Change detection**: Compares new hashes with `data/last_hash.json` to find which specific queues changed
4. **Screenshot capture**: For changed queue(s), `site_content.py` uses Playwright to capture the relevant section from the website
5. **Notification**: `telegram_handler.py` sends a message like "⚡ Для черги 3.1 🔔 ОНОВЛЕННЯ..." with the screenshot attached
6. **User verification**: User views the screenshot or clicks the link to see what actually changed (bot doesn't parse the diff)

## Project structure

.
├── monitor.py # Main orchestrator script
├── site_content.py # API calls + Playwright screenshot capture
├── telegram_handler.py # Telegram Bot API wrapper
├── log_utils.py # Log buffer and formatting
├── data/
│ ├── current.json # Latest API data (all queues)
│ └── last_hash.json # Per-queue hashes
├── .github/
│ └── workflows/
│ └── monitor.yml # GitHub Actions cron job
├── requirements.txt # Python dependencies
└── README.md

text

### Key modules

- **`monitor.py`**: Entry point; fetches API data, computes per-queue hashes, detects changes, triggers notifications
- **`site_content.py`**: Handles API requests and Playwright-based screenshot capture between specific elements on the page
- **`telegram_handler.py`**: Sends messages and images to Telegram channels (main + log)
- **`log_utils.py`**: Buffers log entries with timestamps (Kyiv timezone) and flushes to log channel at end of run

## Setup

### 1. Environment variables

Create a `.env` file or set GitHub Secrets with:

TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_main_channel_id
TELEGRAM_LOG_CHAT_ID=your_log_channel_id
API_ENDPOINT=https://example.com/api/queues
WEBSITE_URL=https://example.com/schedule

text

### 2. Install dependencies

pip install -r requirements.txt
playwright install chromium

text

### 3. Run locally

python monitor.py

text

### 4. Deploy to GitHub Actions

The included `.github/workflows/monitor.yml` runs on a cron schedule. Adjust the cron expression and secrets as needed.

## Migration from `sitemonitor-screen`

### What stayed the same
- **GitHub Actions**: Automated scheduling via cron
- **Telegram integration**: Bot notifications + separate log channel
- **State management**: JSON files in `data/` directory for comparison
- **Screenshot capture**: Playwright still used to grab visual context from website
- **Modular design**: Separated concerns (data fetching, notification, logging)

### What changed
- **Data source**: API provides structured data for all queues → easier to iterate and compare
- **Change granularity**: Hash per queue (not one hash for entire page) → know exactly which queue updated
- **Detection logic**: API hash comparison first, screenshot only when change detected → faster
- **Notification content**: Bot tells you *which queue* changed, not *what* changed (user views screenshot/link to see details)

### Why the old approach worked well
The previous version captured screenshots between two text anchors on the page, which cleanly isolated the content block without ads or navigation. It worked perfectly and was reliable. The migration to API happened primarily for speed: API calls are significantly faster than full page rendering with Playwright, and also made it possible to monitor multiple queues simultaneously and identify precisely which one changed.

## Example notification

```The bot sends a message like:

⚡ Для черги 3.1 🔔 ОНОВЛЕННЯ ГРАФІКА

[Screenshot of the changed schedule section]

🔗 Переглянути графік на сайті

Дата оновлення інформації - 16:00 14.12.2025
```
The bot doesn't parse or explain what changed—it just tells you which queue to look at. You view the screenshot or click the link to see the actual changes.


## License

This project is licensed under the MIT License – see the [LICENSE](LICENSE) file for details.



[![Rehina Nanaka profile views](https://u8views.com/api/v1/github/profiles/212413806/views/day-week-month-total-count.svg)](https://u8views.com/github/reginanka)


## Acknowledgments

This project builds on the architecture of [sitemonitor-screen](https://github.com/reginanka/sitemonitor-screen), evolving it to leverage API data for granular per-queue monitoring while maintaining the visual confirmation approach that made the original version reliable.
