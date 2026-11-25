# Fortex Insurance Group AMS (fortex-dashboard)

Guide for any Claude session working on this repo. Read this first.

## Who this is for
Paul Tadrous, owner of Fortex Insurance Group LLC (DBA Fortex Insure, fortexinsure.com), an independent P&C agency in Celina, TX. Paul is not a developer.
- Write complete, ready-to-use code. Never partial snippets.
- Give numbered, plain-English steps for anything Paul must do by hand.
- Budget is $0. Only free tiers (Vercel Hobby, Supabase Free, Google Workspace tools he already has).
- Writing style: never use em dashes or double dashes in anything written for Paul.
- At the start of each session, say which phase we are in (see Roadmap).

## Current stack (as of Nov 2026)
- Front end: single file `index.html` (vanilla JS, no build step). `vercel.json` rewrites every path to it.
- Hosting: Vercel team `fortex-insure`, project `fortex-dashboard` (prj_rDavGEQEiRxJDBwOt5d5BfvMflF8). Every push to `main` deploys to production at fortex-dashboard.vercel.app. Other branches get preview URLs.
- Database and login: Supabase project "Fortex Dashboard" (ref `nuatfftxifgpyjugjqsb`, us-west-2). One user account (Paul).
- Today ALL data lives in one row: table `public.fortex_data`, id `main`, column `data` (jsonb) with arrays: clients, policies, quotes, tasks, reshop, vehicles, properties, expenses, reminders. Every save overwrites the whole row.
- Backup taken before redesign: `backups.fortex_data_20261125` (not exposed to the API).

## Rules for changing things
1. Back up before any data change: `create table backups.fortex_data_YYYYMMDD as select *, now() as backed_up_at from public.fortex_data;`
2. Never push risky changes straight to `main`. Use a branch, test on the Vercel preview URL, then merge.
3. Every new Supabase table needs Row Level Security ON with policies. Run the Supabase security advisor after any schema change.
4. Escape all user-entered text before putting it in innerHTML.
5. Never put secrets or API keys in `index.html`. The Supabase publishable key is fine to be public; service keys are not.
6. Files in this repo other than index.html and vercel.json are excluded from deploys by `.vercelignore`.

## Known issues found in the Nov 2026 audit
- Single JSON blob storage (last save wins across devices, no history).
- `save()` reports "Saved" even when Supabase returns an error (supabase-js resolves with `{error}`, it does not reject).
- Policies past their renewal date drop off Renewals and the home screen instead of being flagged as overdue.
- Commission math uses the new business rate even on renewal terms, and has no aggregator split or paid vs expected tracking.
- Daily Digest "Send" calls the Anthropic API directly from the browser and cannot work.
- User text is inserted into HTML unescaped.
- Supabase RLS policy allows ANY logged-in user; public sign-ups must stay disabled.

## Roadmap (AMS redesign)
- Phase 0: Audit and planning (done Nov 25, 2026)
- Phase 1: Stabilize: fix save errors, overdue renewals, escaping, security settings, one-click backup
- Phase 2: New data model in Supabase (accounts, contacts, leads, quotes, policies, drivers, vehicles, properties, carriers, commissions, tasks, activities, documents) and migration of existing data
- Phase 3: Redesigned UI (Home, Client 360, Leads pipeline, Policy detail, Commissions)
- Phase 4: Leads intake, commission reconciliation, documents and activity log, helper login with limited access
- Phase 5: Automations (renewal and follow-up emails, reminders, scheduled reports)
