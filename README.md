# Maine Police Monitor

**Watches public Facebook pages of Maine police departments, sheriff's offices and state agencies, then has Claude triage and summarize each new post for a reporter in Slack.**

Agencies often post news on Facebook first, sometimes only there: fatal crashes, manhunts, arrests, missing people, road closures. This tool checks those pages every 30 minutes and sorts each new post into one of three groups:

- **Urgent** posts go to Slack right away. These are deaths, shootings, serious crashes, active searches, missing kids or vulnerable adults, lockdowns and major closures.
  > 🚨 **Urgent: crash** in Carmel
  > • **Maine State Police**: Fatal crash closes I-95 northbound near Exit 174 ([post](#), Thu 6:12 a.m.)
  > A tractor-trailer and a car collided at about 5 a.m.; one person died. Northbound lanes are closed.
  > _Follow up:_ Ask MSP for the victim's name and the crash cause.
- **Notable** posts (arrests, drug seizures, fires, scam warnings, updates to earlier incidents) collect into a **digest at 7 a.m. and 3 p.m.**, with a summary, the people named and a suggested follow-up for each.
- **Routine** posts (Coffee with a Cop, recruiting, birthdays, lost dogs) appear only as a one-line count per agency.

Every summary comes from the agency's own post. **Confirm with the agency before publishing.**

## How it works

1. **Apify** ([Facebook Posts Scraper](https://apify.com/apify/facebook-posts-scraper)) fetches only the posts made since the last poll from every page in `pages.json`. Apify, not our IP, deals with Facebook's login walls and blocking.
2. **Claude** (`claude-opus-5-5`, low effort, structured output) triages them in batches. Apify also reads the text in images, which many agencies use for press releases. When a post has little text, Claude sees the first photo too.
3. **Slack** gets the posts through an incoming webhook.
4. **`state.json`** tracks seen post IDs (so nothing is announced twice) and the digest queue.

## Setup

Needs Python 3.10+ (the Anthropic SDK requires it; macOS's built-in `python3` is 3.9).

```bash
uv venv -p 3.11 .venv && uv pip install -p .venv -r requirements.txt
source .venv/bin/activate                  # then `python3` below is the venv's 3.11
cp .env.example .env                       # add APIFY_TOKEN, ANTHROPIC_API_KEY, SLACK_WEBHOOK_URL
python3 monitor.py check-pages             # find dead or wrong page URLs (costs ~8¢)
python3 monitor.py poll --dry-run          # one poll, prints what it would post
python3 monitor.py digest --dry-run        # prints the digest
```

### ⚠️ Check `pages.json` first

The starter list of 39 agencies uses **page URLs I guessed and haven't checked**. Run `check-pages`. For any row marked `NONE` or `STALE`, find the agency's real page in a browser and fix the URL. Add or remove agencies as needed. Each entry needs a `name` and a `url`.

## Running it on a schedule

`.github/workflows/monitor.yml` runs a **poll every 30 minutes** and a **digest at 11:00 and 19:00 UTC**. That's 7 a.m. and 3 p.m. during daylight time, or 6 a.m. and 2 p.m. in winter. Add three repo secrets: `APIFY_TOKEN`, `ANTHROPIC_API_KEY` and `SLACK_WEBHOOK_URL`. You can also run any command by hand from the Actions tab.

Both jobs live in one workflow with a concurrency lock, so they never overwrite each other's `state.json`. State lives in the Actions cache, the same pattern as maine-jet-tracker. If the cache is evicted, the next poll looks back `FIRST_LOOKBACK_HOURS` (24) and could repeat some posts. Nothing gets lost.

For cron on a droplet instead:

```cron
*/30 * * * * cd /path/to/maine-police-monitor && python3 monitor.py poll   >> monitor.log 2>&1
0 7,15 * * * cd /path/to/maine-police-monitor && python3 monitor.py digest >> monitor.log 2>&1
```

## Cost

- **Apify:** about $2 per 1,000 posts. Each poll asks only for posts since the last poll, so ~40 agencies posting a few times a day costs **roughly $5–10/month**.
- **Claude:** the same posts in batches of 20 at low effort cost **a few dollars a month**.
- Short polling intervals cost almost nothing extra, because Apify charges per post returned, not per run.

## Health checks

The digest flags pages that failed to load on the last poll. It also warns if no poll has succeeded in 6 hours. If a digest says "No new posts" for days, Apify or the token is the likely cause.

## Caveats

- Scraping Facebook through Apify is a gray area under Facebook's terms. It collects only public posts from government agencies, but know that going in.
- Claude summarizes only what the post says. It can still misread a post, so the link to the original is always included.
- Names come straight from police posts. Follow BDN policy on naming people who are charged or are juveniles.
