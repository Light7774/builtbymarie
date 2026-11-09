# builtbymarie.com: live automation playground

My portfolio site for automation services. Visitors don't just read about what I build.
They **try a real automation on their own business**, right on the page, and get the result in seconds.

Live site: https://www.builtbymarie.com

## What's in it

| Demo | What the visitor does | What the automation does | Status |
|---|---|---|---|
| Win-back sequence generator | Describes their membership and why members leave | Writes a 3-email win-back sequence, shows it on the page, emails a copy, logs the lead, notifies me | Building |
| SOP generator | Pastes messy notes or a call transcript | Returns a clean, step-by-step SOP | Planned |
| Feedback miner | Pastes reviews or survey answers | Groups them into themes and flags churn signals | Planned |

## How it works

```
Static site (Cloudflare Pages)
   │  fetch POST (JSON)
   ▼
n8n webhook (self-hosted, Docker + Traefik on a VPS)
   ├─ validate input, honeypot, rate limit
   ├─ LLM call (Groq) → structured JSON
   ├─ respond to the page
   └─ Google Sheets log · email copy · Telegram alert
```

## Repo layout

```
site/            the website (plain HTML/CSS/JS, no build step)
n8n/workflows/   exported n8n workflows (credentials stripped)
docs/            build guides and notes
```

## Run the site locally

Open `site/index.html` in a browser, or serve the folder:

```
npx serve site
```

The demo form posts to the production webhook set in the `WEBHOOK` constant inside `site/index.html`.

## Security notes

- No API keys or credentials live in this repo. They stay in n8n's credential store.
- Workflow exports are checked for secrets before each commit.
- Every demo is rate-limited per email and per day.

## Author

Marie Angelie Gerios · AI automation and operations · angelie@builtbymarie.com
