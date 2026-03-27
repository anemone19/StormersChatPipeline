# StormersChatPipeline

Scheduled ETL pipeline for Stormers Rugby data.

This repository contains the standalone Posit Connect deployable that pulls Oval Insights API data into Neon PostgreSQL using [run_pipeline.Rmd](run_pipeline.Rmd).

## Files

- `run_pipeline.Rmd`: ETL entrypoint for Posit Connect scheduled runs
- `manifest.json`: Posit Connect manifest for the pipeline content root
- `backend/.env.example`: local environment variable template
- `scripts/create_tables.sql`: database schema for the target tables
- `context/deployment_checklist.md`: publish and scheduling runbook
- `context/download_pipeline_context.md`: table and field reference

## Local setup

1. Copy `backend/.env.example` to `backend/.env`
2. Fill in the Neon and Oval Insights credentials
3. Open `run_pipeline.Rmd` in RStudio and render it

## Posit Connect

Import this repository from GitHub and set the primary file to `run_pipeline.Rmd`.
Then add the required environment variables in the Posit Connect Vars panel and enable the schedule.
