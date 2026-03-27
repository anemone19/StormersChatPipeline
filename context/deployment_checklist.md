# StormersChat — Posit Connect Deployment Checklist
*Use this to deploy and schedule the ETL pipeline on Posit Connect Cloud, then verify the Shiny app is pointing at the refreshed database.*

> Security note: use placeholders in docs and store the real values only in `backend/.env` or Posit Connect Vars. If raw secrets were previously committed, rotate them.

---

## Step 1 — Generate the pipeline manifest

In RStudio, run in the Console:

```r
rsconnect::writeManifest("pipeline")
```

This creates `manifest.json`. Then commit and push:

```bash
git add manifest.json
git commit -m "add pipeline manifest for Posit Connect"
git push
```

---

## Step 2 — Deploy the pipeline to Posit Connect

1. Log in to [connect.posit.cloud](https://connect.posit.cloud)
2. Click **Publish** in the sidebar → **Import from GitHub**
3. Select your repository and branch (`main`)
4. Set **Primary file** to `run_pipeline.Rmd`
5. Click **Publish**

> If RStudio isn't yet connected to Posit Connect, run this first:
> ```r
> rsconnect::addConnectServer("connect.posit.cloud")
> ```
> A browser window will open — log in with your Posit account.

---

## Step 3 — Set pipeline environment variables on Posit Connect

In the deployed pipeline → **⚙ Settings → Vars**, add all of these:

```
OVAL_INSIGHTS_BASE_URL        = https://api.rugbyviz.com/rugby/v1
OVAL_INSIGHTS_USERNAME        = <your_oval_insights_username>
OVAL_INSIGHTS_PASSWORD        = <your_oval_insights_password>
OVAL_URC_COMP_ID              = 1068
OVAL_CHAMPIONS_CUP_COMP_ID    = 1008
OVAL_CURRENT_SEASON_ID        = 202501
OVAL_STORMERS_TEAM_ID         = 3994

PG_HOST                       = <your_neon_host>
PG_PORT                       = 5432
PG_DBNAME                     = <your_database_name>
PG_USER                       = <your_database_user>
PG_PASSWORD                   = <your_database_password>
```

Save — the pipeline will restart automatically.

---

## Step 4 — Run the pipeline once manually

Before enabling the schedule, run the pipeline once from Posit Connect:

1. Open the deployed `run_pipeline.Rmd` content item
2. Click **Run** or **Render now**
3. Wait for the report to finish successfully

Use the rendered report plus direct SQL checks to verify row counts. Expected ballpark counts:

| Table | Expected |
|---|---|
| matches | ~200+ |
| player_match_stats | ~7,000+ |
| match_team_stats | ~300+ |
| match_events | ~300,000+ |
| match_quarters | ~8,000+ |
| head_to_head | ~10+ (Stormers matches only) |
| standings | ~40 |
| round_standings | ~300+ |
| season_stats | ~40 |
| squad | ~1,700+ |
| venues | ~40+ |
| officials | ~180+ |
| player_top10 | ~600+ |
| team_top_stats | ~1,300+ |

If the report looks inconsistent, verify directly in Neon:

```sql
SELECT 'player_match_stats' AS table_name, COUNT(*) FROM player_match_stats
UNION ALL
SELECT 'match_team_stats', COUNT(*) FROM match_team_stats
UNION ALL
SELECT 'match_events', COUNT(*) FROM match_events
UNION ALL
SELECT 'match_quarters', COUNT(*) FROM match_quarters
UNION ALL
SELECT 'head_to_head', COUNT(*) FROM head_to_head;
```

If any table is unexpectedly low or zero, check the render logs for `✗ Failed:` messages or schema errors before scheduling it.

---

## Step 5 — Enable the daily schedule

In the deployed pipeline → **⚙ Settings → Schedule**:

- Enable scheduling: **On**
- Frequency: **Daily**
- Time: **06:00 SAST** (04:00 UTC)

This runs every morning before the coaching staff arrive, picking up any matches
that finalised overnight.

---

## Step 6 — Update the Shiny app environment variables

The Shiny app (`apps/chat`) is already deployed but may still have older database Vars.
Update them to the current Neon values:

In the deployed Shiny app → **⚙ Settings → Vars**, update or add:

```
PG_HOST      = <your_neon_host>
PG_PORT      = 5432
PG_DBNAME    = <your_database_name>
PG_USER      = <your_database_user>
PG_PASSWORD  = <your_database_password>
```

Leave these unchanged (already set):
```
ANTHROPIC_API_KEY   = (your key)
CLAUDE_MODEL        = claude-sonnet-4-6
APP_USERNAME        = stormers
APP_PASSWORD        = (your password)
```

Save — the app restarts automatically and will now query Neon.

---

## Step 7 — Smoke test the chat app

Open the Shiny app URL and try these queries in order:

1. **"Who leads the URC?"** — tests `standings` table
2. **"What was our last result?"** — tests `matches` table + date casting
3. **"Who are our top try scorers?"** — tests `player_top10`
4. **"What is our lineout success rate?"** — tests `match_team_stats` JOIN
5. **"Who scored our tries against Leinster?"** — tests `match_events` with lowercase event_type

If any query returns "couldn't find any data" or a SQL error, check:
- The Vars panel has all 5 PG_ variables set correctly
- The pipeline ran successfully and the database counts look right
- The SQL shown in the "Show SQL" expander looks sensible

---

## Redeployment (ongoing)

After any code change to `app.R`:

```bash
# Only needed if you installed new R packages:
# rsconnect::writeManifest("apps/chat")

git add apps/chat/app.R
git commit -m "describe change"
git push
```

Posit Connect redeploys automatically on push if "Automatically publish on push" is enabled.

After any code change to `run_pipeline.Rmd`:

```bash
rsconnect::writeManifest(".")   # always regenerate
git add run_pipeline.Rmd manifest.json
git commit -m "describe change"
git push
```
