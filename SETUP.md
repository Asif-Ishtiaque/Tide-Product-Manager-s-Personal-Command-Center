# Connecting Jira + Figma to your dashboard

The dashboard can't call Jira/Figma directly (browsers are blocked by CORS).
The `worker.js` proxy runs on Cloudflare's free tier, holds your tokens, and
returns one clean payload. Total time: ~20 minutes.

> Your tokens live ONLY as Cloudflare secrets. They never go in the dashboard,
> in the code, or in any URL. Don't commit them to git.

---

## 1. Get your tokens (only you can do this)

**Jira API token**
1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. "Create API token", name it, copy it somewhere safe for a minute.
3. Note your Atlassian email and your site (e.g. `yourcompany.atlassian.net`)
   and your project key (the prefix on issues, e.g. `PAY` in `PAY-412`).

**Figma personal access token**
1. Figma → your avatar → Settings → Security → Personal access tokens.
2. Generate one with at least *File content: read* and *Comments: read*.
3. Grab the file keys you want on the board. A Figma URL looks like
   `figma.com/file/ABC123xyz/My-Design` → the key is `ABC123xyz`.

---

## 2. Deploy the Worker (free)

Install the CLI and log in:
```bash
npm install -g wrangler
wrangler login
```

Create the project and drop in the file:
```bash
mkdir pm-proxy && cd pm-proxy
npm create cloudflare@latest . -- --type=hello-world
# replace the generated src/index.js with worker.js (contents from this build)
```

Set your secrets (you'll be prompted to paste each value — nothing is stored in code):
```bash
wrangler secret put JIRA_SITE         # yourcompany.atlassian.net
wrangler secret put JIRA_EMAIL        # you@company.com
wrangler secret put JIRA_TOKEN        # the Atlassian token
wrangler secret put JIRA_PROJECT      # PAY
wrangler secret put FIGMA_TOKEN       # the Figma token
wrangler secret put FIGMA_FILES       # ABC123xyz,DEF456uvw

# optional — enables the sprint widget:
wrangler secret put JIRA_BOARD_ID     # numeric Scrum board id
wrangler secret put JIRA_POINTS_FIELD # e.g. customfield_10016 (story points)

# optional — lock down who can read the proxy:
wrangler secret put ALLOW_ORIGIN      # https://your-dashboard-url
```

Deploy:
```bash
wrangler deploy
```
Copy the URL it prints, e.g. `https://pm-proxy.you.workers.dev`.

---

## 3. Point the dashboard at it

Open `pm-command-center.html` and edit the CONFIG block at the top of the script:
```js
const CONFIG = {
  mode: 'live',                                   // was 'mock'
  proxyUrl: 'https://pm-proxy.you.workers.dev',   // your Worker URL
  refreshMs: 60000
};
```
Save, open the file (or your hosted copy). The badge at the bottom should read
**LIVE** and the board fills with your real data. The refresh button and the
60-second auto-refresh both pull live now.

---

## Troubleshooting
- **Still shows mock / badge says MOCK** — `mode` isn't `'live'`, or the proxy
  URL is wrong. Open the browser console; a `Live fetch failed` line points you.
- **Sprint widget says "No active sprint"** — set `JIRA_BOARD_ID`, or your board
  isn't a company-managed Scrum board (team-managed sprints aren't exposed the
  same way).
- **Jira 404 on `/search/jql`** — older instance: in `worker.js`, change the
  endpoint to `/rest/api/3/search` (the request body is identical).
- **Status names don't match** — your team may use "Doing"/"On Hold" instead of
  "In Progress"/"Blocked". Tweak the JQL strings in `getJira()`.
- **Empty Figma section** — token missing comment/file-read scope, or the file
  keys are wrong.
