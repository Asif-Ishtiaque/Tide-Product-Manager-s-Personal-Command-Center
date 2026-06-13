# Tide

A personal ops dashboard that unifies Jira × Figma × ClickUp × Slack into one
prioritized attention feed. Two parts:

```
tide/
├── dashboard/index.html   ← the board (static, runs in any browser)
├── proxy/                 ← Cloudflare Worker that holds your tokens
│   ├── worker.js
│   ├── wrangler.toml
│   ├── package.json
│   └── .dev.vars.example  ← copy to .dev.vars, add your tokens
└── SETUP.md               ← token + deploy reference
```

The dashboard can't call Jira/Figma directly (CORS). The proxy calls them
server-side and returns one JSON payload the dashboard renders.

---

## Run it in Antigravity (or any VS Code-based IDE)

**Prerequisite:** Node.js 18+ (`node -v` in the terminal to check).

### Option A — let the Antigravity agent do it
Open this folder in Antigravity, then paste this into the agent:

> Set up and run this project locally.
> 1. In `proxy/`, copy `.dev.vars.example` to `.dev.vars` and pause so I can fill in my tokens.
> 2. After I confirm, run `npm install` then `npx wrangler dev` in `proxy/` and tell me the local URL it prints (usually http://localhost:8787).
> 3. In `dashboard/index.html`, set CONFIG.mode to 'live' and CONFIG.proxyUrl to that local URL.
> 4. Serve the dashboard with `npx serve dashboard` and give me the URL to open.
> Do not put my tokens anywhere except `.dev.vars`.

### Option B — do it yourself in the integrated terminal
```bash
# 1. proxy: add your tokens locally
cd proxy
cp .dev.vars.example .dev.vars        # then edit .dev.vars with your tokens
npm install
npx wrangler dev                      # prints http://localhost:8787
```
Open `dashboard/index.html` and edit the CONFIG block:
```js
const CONFIG = { mode: 'live', proxyUrl: 'http://localhost:8787', refreshMs: 60000 };
```
```bash
# 2. serve the dashboard (new terminal tab)
cd ..
npx serve dashboard                   # open the printed localhost URL
```
The bottom badge flips to **LIVE** and the board fills with your data.

---

## Going live for real (not just localhost)
When local works, deploy the proxy so it's always on:
```bash
cd proxy
npx wrangler login
# set the same values as secrets (prompted, never stored in code):
npx wrangler secret put JIRA_SITE
npx wrangler secret put JIRA_EMAIL
npx wrangler secret put JIRA_TOKEN
npx wrangler secret put JIRA_PROJECT
npx wrangler secret put FIGMA_TOKEN
npx wrangler secret put FIGMA_FILES
npx wrangler deploy                   # prints https://tide-proxy.<you>.workers.dev
```
Point `CONFIG.proxyUrl` at that URL. Host `dashboard/` free on Cloudflare Pages,
Netlify Drop, or GitHub Pages. See SETUP.md for token sources + troubleshooting.

> Tokens live only in `.dev.vars` (local) or Wrangler secrets (deployed).
> Never commit them; `.gitignore` already excludes `.dev.vars`.
