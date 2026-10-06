# Maine Police Monitor

**Watches public Facebook pages of Maine police departments, sheriff's offices and state agencies, then has Claude triage and summarize each new post for a reporter in Slack.**

Agencies often post news on Facebook first, sometimes only there: fatal crashes, manhunts, arrests, missing people, road closures. This tool checks those pages every 10 minutes and sorts each new post into one of three groups:

- **Urgent** posts go to Slack right away. These are deaths, shootings, serious crashes, active searches, missing kids or vulnerable adults, lockdowns and major closures.
  > 🚨 **Urgent: crash** in Carmel
  > • **Maine State Police**: Fatal crash closes I-95 northbound near Exit 174 ([post](#), Thu 6:12 a.m.)
  > A tractor-trailer and a car collided at about 5 a.m.; one person died. Northbound lanes are closed.
- **Notable** posts (arrests, drug seizures, fires, scam warnings, updates to earlier incidents) collect into a **digest at 7 a.m. and 3 p.m.**, with a summary and the people named for each.
- **Routine** posts (Coffee with a Cop, recruiting, birthdays, lost dogs) appear only as a one-line count per agency.

Every summary comes from the agency's own post. **Confirm with the agency before publishing.**

## How it works

1. **Apify** ([alfalfa/facebook-posts-scraper](https://apify.com/alfalfa/facebook-posts-scraper)) fetches recent posts from every page in `pages.json`. Each poll re-checks the last 30 minutes, because a single check occasionally misses a post. Posts already seen are skipped by ID. Apify, not our IP, deals with Facebook's login walls and blocking.
2. **Claude** (`claude-opus-5-5`, low effort, structured output) triages them in batches. Apify also reads the text in images, which many agencies use for press releases. When a post has little text, Claude sees the first photo too.
3. **Slack** gets the posts from the **Police Pages** bot (app `A0C5XJRVB1T`, defined in `manifest.json`). It needs `SLACK_BOT_TOKEN` and `SLACK_CHANNEL`. An incoming-webhook URL (`SLACK_WEBHOOK_URL`) also works instead.
4. **`state.json`** tracks seen post IDs (so nothing is announced twice) and the digest queue.

## Setup

Needs Python 3.10+ (the Anthropic SDK requires it; macOS's built-in `python3` is 3.9).

```bash
uv venv -p 3.11 .venv && uv pip install -p .venv -r requirements.txt
source .venv/bin/activate                  # then `python3` below is the venv's 3.11
cp .env.example .env                       # add APIFY_TOKEN, ANTHROPIC_API_KEY, SLACK_BOT_TOKEN, SLACK_CHANNEL
python3 monitor.py check-pages             # find dead or wrong page URLs (costs ~8¢)
python3 monitor.py poll --dry-run          # one poll, prints what it would post
python3 monitor.py digest --dry-run        # prints the digest
```

### Editing `pages.json`

Each entry needs a `name` and a `url`. After any change, run `check-pages`, which costs about 7¢. Watch for out-of-state lookalike pages: `SanfordPolice` turned out to be Sanford, Florida, and `FranklinCountySheriff` was Virginia. Pages not monitored yet because no working URL was found: Auburn PD, Knox County Sheriff, Lincoln County Sheriff and Brunswick PD.

## Running it on a schedule

`.github/workflows/monitor.yml` runs **one long job**, `monitor.py loop`, that polls every **10 minutes** (`POLL_SECONDS`). After about 5½ hours it saves state and starts its own successor. GitHub's cron is too unreliable to use directly: a `*/5` schedule actually ran about 4 times a day. The hourly cron in the workflow is only a backstop that restarts the loop if a handoff fails.

Each poll also sends the digest when one is due. Digests go out at **7 a.m. and 3 p.m. Eastern** all year (`DIGEST_HOURS_ET`). If a digest slot is missed by more than 3 hours, those items roll into the next digest. You can run `poll`, `digest` or `check-pages` by hand from the Actions tab. To stop everything, disable the workflow.

The repo is public, so Actions minutes are free; a private repo would burn through the monthly allowance with a job running around the clock. Keys are stored as repo secrets: `APIFY_TOKEN`, `ANTHROPIC_API_KEY`, `SLACK_BOT_TOKEN` and `SLACK_CHANNEL`. Run logs are public, but they contain only headlines from public posts.

**Lag:** an urgent post reaches Slack about 10 minutes after it goes up, on average 5. Lower `POLL_SECONDS` to poll more often (see Cost).

State (seen post IDs and the digest queue) lives in the Actions cache. If the cache is evicted, the next poll looks back `FIRST_LOOKBACK_HOURS` (24) and could repeat some posts.

### The Slack app

The bot's name, scopes (`chat:write`, `chat:write.public`) and description live in `manifest.json`. To change them, edit the file and run `slack manifest sync` (Slack CLI). Find the bot token at https://api.slack.com/apps/A0C5XJRVB1T/oauth.

## Cost

Measured 2026-10-02, not estimated:

- **Apify** bills **$0.002 per post returned**. A check that finds nothing costs **$0**, with no per-run or per-page fee. The bill depends on how many posts the agencies publish (about 15–20 a day across 35 pages) and on how often each post is re-returned within the 30-minute lookback, about 3 times at 10-minute polling. That's **about $4–5 a month**, inside Apify's free $5 credit. 5-minute polling is about $8–10 a month and needs a paid plan.
- **Limits:**
  - The free plan stops at $5 a month and can't bill more; runs simply fail until the 1st.
  - Each run is capped at `APIFY_MAX_USD_PER_RUN` (25¢) on top of that.
- **Don't switch back to `apify/facebook-posts-scraper` casually.** It bills about $0.007 per page on every check, even an empty one. That's about $0.25 per poll, or around $2,000 a month at 5-minute polling. It's kept only as a fallback (`APIFY_ACTOR=apify~facebook-posts-scraper`).
- **Claude** triages new posts in batches at low effort, which costs **a few dollars a month**.

## Health checks

The digest flags pages that failed to load on the last poll. It also warns if no poll has succeeded in 6 hours. If a digest says "No new posts" for days, Apify or the token is the likely cause.

## Caveats

- Scraping Facebook through Apify is a gray area under Facebook's terms. It collects only public posts from government agencies, but know that going in.
- Claude summarizes only what the post says. It can still misread a post, so the link to the original is always included.
- Names come straight from police posts. Follow BDN policy on naming people who are charged or are juveniles.
