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

## Phase 1 (live Nov 25, 2026): how things work now
- Saving: `save()` calls RPC `public.save_fortex_data(p_data, p_expected_version)`. A version mismatch raises VERSION_CONFLICT and the app shows a red banner (reload / download copy / keep mine). Other errors show a banner and auto-retry every 15s. If loading fails the app refuses to save.
- History: trigger `trg_fortex_history` copies the previous row into `backups.fortex_data_history` on every change (30 days). pg_cron job `fortex-nightly-backup` (08:00 UTC) snapshots into `backups.fortex_data_nightly` (60 days).
- Access: RLS policy `fortex_data_app_users` = `private.is_app_user()` (table `public.app_users`) AND `private.mfa_ok()` (requires aal2 once a verified TOTP factor exists). Add a future user by inserting into `app_users`.
- UI helpers: `esc()` for all user text, `appConfirm()` / `appAlert()` in-app dialogs (never use browser alert/confirm), forms do not close on outside click, `closeModal()` resets edit state.
- Renewals: Overdue tab and Home card; Renew Policy box (`renewPolicy`/`saveRenewal`) logs a note to the client; `commRateFor()` uses the renewal rate when commType is Renewal; `isLive()` excludes Cancelled/Non-Renewed/Lapsed.
- Security and Backup page: TOTP two-factor setup and full JSON backup download.

## Statement catch-up and commission tables (Nov 25, 2026)
- Loaded every commission statement Feb to Oct 2026: Smart Choice ($7,121.09 paid, ties to pay stub YTD), First Connect ($5,203.75), Progressive Direct ($1,495.46).
- Tables (RLS same as fortex_data): `carrier_commission_rates` (unique channel+carrier+line_of_business+transaction; gross_rate, agency_share, net_rate), `commission_statements` (statement_key = source|YYYY-MM| prevents double loads, total_paid = what Paul actually received), `commission_lines` (PK statement_key+line_no; kind = commission, bonus, fee, carry_forward; client_id/policy_id link to fortex_data ids).
- Smart Choice share: 70% to Paul on Smart Start (SS) and Agent Program (AP); 100% on Express and adjustments. First Connect and Progressive Direct pay Paul 100%.
- Policies now carry `channel` (Smart Choice, First Connect, Progressive Direct, AIM, Direct Appointment) and imported ones carry `source`. Clients from EZLynx carry `ezlynxId` (client id = 'c' + EZLynx applicant id).
- App: Carrier Rates page, policy form auto-fills net rates via findCarrierRate(), learnCarrierRate() saves new carrier rates, Commissions page shows Commission Received from commission_statements.
- To load a new month: parse statement, insert into commission_statements then commission_lines with on conflict do nothing, link policy_id by normalized policy number (alphanumeric uppercase).

## Carrier picker (Nov 25, 2026)
- Table `public.carriers` (name, name_key unique = lowercase alphanumeric, default_channel, note). RLS same as fortex_data, anon revoked.
- Policy form and Carrier Rates form use a custom dropdown `initCombo()` (no native datalist). Picking a carrier sets Written Through to `carrierDefaultChannel()` then runs `applyCarrierRate()`. "+ Add" saves a new carrier with its usual channel via `ensureCarrier()`; savePolicy also calls ensureCarrier.
- Carriers can have one row per channel: unique (name_key, default_channel), `is_primary` marks the default. Dropdown shows each pair (e.g. Progressive via Progressive Direct, Progressive via Smart Choice); picking a pair sets that channel. Saving a policy with a known carrier through a new channel adds a non-primary row.
- Progressive defaults to Progressive Direct (new Progressive business goes direct); older renewing Progressive policies are via Smart Choice.

## One-time bonus commission (Nov 25, 2026)
- Policy fields `bonusRate` (%) and `bonusNote`. `bonusAmt(p)` = premium x bonusRate for the current term only; NOT included in commAmt/commAmtAnnual and never written to carrier_commission_rates.
- Shown as "+ $X one-time bonus" (`bonusTag`) on policy lists and a "One-Time Bonuses" card on Commissions. `saveRenewal` moves the bonus into `bonusHistory` and clears it.

## Still open (later phases)
- Single JSON blob storage (fixed by the Phase 2 data model).
- No aggregator split or paid vs expected commission tracking (Phase 4).

## Roadmap (AMS redesign)
- Phase 0: Audit and planning (done Nov 25, 2026)
- Phase 1: Stabilize (done Nov 25, 2026)
- Phase 2: New data model in Supabase (accounts, contacts, leads, quotes, policies, drivers, vehicles, properties, carriers, commissions, tasks, activities, documents) and migration of existing data
- Phase 3: Redesigned UI (Home, Client 360, Leads pipeline, Policy detail, Commissions)
- Phase 4: Leads intake, commission reconciliation, documents and activity log, helper login with limited access
- Phase 5: Automations (renewal and follow-up emails, reminders, scheduled reports)
