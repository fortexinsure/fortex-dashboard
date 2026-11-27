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
- Data lives in one table per record type (Phase 2, Nov 25, 2026): `clients`, `policies`, `quotes`, `tasks`, `reshop`, `vehicles`, `properties`, `expenses`, `reminders`. `public.fortex_data` (id `main`) is kept as an automatic mirror of all of them (see Phase 2 section).
- Backup taken before redesign: `backups.fortex_data_20261125` (not exposed to the API).

## Rules for changing things
1. Back up before any data change: `create table backups.fortex_data_YYYYMMDD as select *, now() as backed_up_at from public.fortex_data;`
2. Never push risky changes straight to `main`. Use a branch, test on the Vercel preview URL, then merge.
3. Every new Supabase table needs Row Level Security ON with policies. Run the Supabase security advisor after any schema change.
4. Escape all user-entered text before putting it in innerHTML.
5. Never put secrets or API keys in `index.html`. The Supabase publishable key is fine to be public; service keys are not.
6. Files in this repo other than index.html and vercel.json are excluded from deploys by `.vercelignore`.

## Phase 1 (live Nov 25, 2026): how things work now
- Saving (since Phase 2): `save()` sends only changed records via `save_book_changes` (see Phase 2 section). The old RPC `save_fortex_data` still works for old tabs. A conflict raises VERSION_CONFLICT and the app shows a red banner (reload / download copy / keep mine). Other errors show a banner and auto-retry every 15s. If loading fails the app refuses to save.
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

## Phase 2 record tables (Nov 25, 2026)
- Each table: `id` text PK, `rev` (bumped on every change by trigger `trg_rev`), `pos` (list order), typed columns, `extra` jsonb for any field without a column (e.g. dlClass), created_at/updated_at. RLS = is_app_user() AND mfa_ok(); anon revoked.
- `private.book_map(tbl, js_key, col, typ)` maps dashboard field names to columns (typ text/numeric/int/date/jsonb/fk). Values that do not fit the column type are kept as typed in `extra`, so nothing is lost. To add a new field: add a column and a book_map row; until then it lands in `extra` automatically.
- `private.rec_to_cols` / `private.row_to_rec` convert both ways (empty numerics/dates come back as '', 0 as number 0, other numbers as strings like the app stores them).
- App: `load_book()` returns {data, revs, version}. `save_book_changes(p_changes, p_force)` takes [{t, op:'upsert'|'delete', id, rec, rev, pos}], checks each record's rev and raises VERSION_CONFLICT (detail lists records) if another device changed it; p_force skips the check ("Keep my version"). The app diffs `data` against `syncedRecs` (collectChanges) so only changed records are sent.
- Mirror: after any table change `private.rebuild_blob()` rewrites fortex_data.data from the tables (version bumps, so other screens notice). If anything writes fortex_data directly (old dashboard tab, save_fortex_data), trigger `trg_blob_to_tables` applies only the records that changed to the tables. Guard: session setting `fortex.syncing='1'` stops the two triggers from looping.
- History: per-record old versions in `backups.record_history` (tbl, rec_id, op, old_row, changed_at, changed_by). No automatic cleanup yet (add a cron job when it grows). Whole-book history and nightly snapshots still run on fortex_data.
- FKs to clients are DEFERRABLE INITIALLY DEFERRED: policies/tasks/vehicles/properties/reminders cascade on delete; quotes/reshop set null.

## Phase 3 screens (Nov 26, 2026)
- Mockups: design canvas https://claude.ai/artifact/R1zcLeFShaAukxfjAR1S3a (Home, Client 360, computer + phone). Paul's choices: refine current navy/gold look, phone matters, mockups first.
- Home (`renderDashboard`, `p3Items`): global search (`p3Search`: names, phones, emails, EZLynx id, policy numbers, VINs; "/" focuses it), quick add buttons, KPI cards (premium on books = live annualized premium, commission received this year from commission_statements, renewals next 60 days, open leads = open quotes + Prospect clients), "Needs you today" (overdue/30-day renewals, tasks due, quote follow-ups, policies with no commission rate, previous month's statements missing after the 15th) with filters, This month, renewal calendar, commission by channel, lead pipeline. Re-renders after loadCarrierRates.
- Client 360 = page `page-client` (no more slide-in panel). `openProfile()` fills it; `c360Mount()` moves the old panel's tab sections (ptab-details/policies/vehicles/property/notes/driver) into `#c360-body` so all their edit functions still work. New tabs: ptab-overview (`refreshC360`), ptab-documents (Google Drive search link until Phase 4), ptab-commission (`loadC360Commission` reads commission_lines by client_id). `closeProfile()` returns to `lastPage`. refreshCurrentPage maps 'client' to refreshC360.
- Phones (<=768px): sidebar hidden, bottom bar `#p3-tabbar` (Home, Clients, + sheet, Tasks, More opens sidebar drawer).
- Leads and Quotes (`renderQuotes`): kanban over Q_STAGES (New Lead, Quoted, Pending Docs, Closed Won, Closed Lost) with HTML5 drag/drop (`dropQuote`), arrows (`moveQuote`), Won/Lost buttons. `setQuoteStage` records `history`, `stageSince`, `wonOn`; moving to Lost always opens `lost-modal` (`openLost`/`saveLost`: lostReason, lostOn, xDate, optional re-quote task, requoteOn). Re-open, Make client (`quoteToClient` creates client + links clientId), filters (search, line, source, show lost), lost reasons chart, Prospects tab (clients tagged Prospect, Start quote links clientId via `quoteClientId`). Quote fields added: currentCarrier, xDate, quotedCarriers, created. deleteQuote now confirms.
- Policy detail = page `page-policy` (`openPolicy`, `refreshPolicyPage`, `closePolicy` returns to the page or client it came from): header with Renew/Edit/Re-shop (`policyReshop` prefills reshop modal)/Mark cancelled (`policyCancel`)/Delete, commission card with statement lines from commission_lines by policy_id, details, client, related vehicles or properties, other policies. Policies list, Renewals list, Home renewal items, Client 360 policy titles and Commissions rows open it.
- Commissions (`renderCommissions`): KPIs, received by month stacked bars by source, by carrier expected/yr vs received this year (commission_lines loaded once per session by `loadCommLinesYear`, matched with carrierKey), by line of business, every active policy. Only active policies count.
- `du()` now parses YYYY-MM-DD as local date (was a day early in Texas).
- Premium rule everywhere (Paul's decision): book size = annual premium of active policies (`annualPrem`: 6 month terms doubled). Book of Business also shows premium in force (current terms of active policies) under the total.

## Phase 4 (in progress, Nov 26, 2026)
- Website leads: Edge Function `lead-intake` (verify_jwt off, only accepts posts from fortexinsure.com) calls `public.intake_web_lead(p)` (service_role only). Creates a quote with source Website, status New Lead, due today. Dedupes same email/phone within 10 min, caps 30 per hour, honeypot fields ignored. Optional secrets: FORMSPREE_URL (forwards for the email alert), LEAD_THANKS_URL. Home shows "New web lead" items.
- Reconciliation: Commissions page has Overview and Reconcile tabs (`reconRows`, `renderRecon`). Rule (Paul): flag Missing when no payment 60+ days after a term starts (only for months whose statements are loaded), Short or Overpaid when off by more than $5. Also lists statement lines with no policy match. Home item and policy page chip use the same rows. Lines load once per session (`loadAllLines`).
- Activity timeline: Client 360 Activity tab (`loadC360Activity`, `renderC360Activity`) merges notes, quote stage history, commission payments and record changes from `public.client_activity(p_client_id)` (reads backups.record_history, app users only).
- Documents: Client 360 Documents tab (`renderC360Docs`) stores Google Drive links on the client (`driveFiles` [{name,url,policyId,added}], `driveFolder`, both in extra). Only drive.google.com and docs.google.com links are accepted. Nov 27: 59 clients auto-linked to their folders under Drive "Customer's Documents" (and its "Commercial" subfolder) by name match; backup backups.fortex_data_20261127_drive. Kirolos Aziz left unlinked (folder "Aziz/Jaklin/Kirolos Mikhail" may not be him).
- Website form: Tally form "Get Your Free Insurance Quote" (formId 5XxGEZ, edit at https://tally.so/forms/5XxGEZ/edit), built and published Nov 27 (public link https://tally.so/r/5XxGEZ). Pages: what to quote (checkboxes), about you, vehicles (if Auto), home (if Home or Renters), business (if Business), current coverage with optional dec page upload (PDF/images, up to 5 files, 10 MB each on free plan). Tally emails paul@fortexinsure.com each submission. A Tally webhook posts to lead-intake, which accepts only formId 5XxGEZ, requires a valid Tally-Signature HMAC (signing secret set on Tally webhook 3yDlNx and in the function; Paul has the phrase), and creates the lead with currentCarrier, xDate and webFiles (dec page links, shown in the quote form as "Documents from the website form"). Advanced styling (radius, sizes) needs Tally Pro so it is not used. Auto extraction of dec page data is not done (would need a paid AI service). Embedded on fortexinsure.com Nov 27: the WordPress home page (post 65, Elementor) is one big HTML widget (data-id dcf7383); the old Formspree form inside div.quote-form was replaced with the Tally embed (no transparent background, white card). Formspree is no longer used. Thank-you page has phone (469) 602-6272 and Facebook/Instagram links; Paul added the navy logo himself.
- Helper login: waiting until Paul hires.

## Still open (later phases)
- Remove the fortex_data mirror once nothing depends on it; drivers, activities, documents tables come with Phase 4 features.
- No aggregator split or paid vs expected commission tracking (Phase 4).

## Roadmap (AMS redesign)
- Phase 0: Audit and planning (done Nov 25, 2026)
- Phase 1: Stabilize (done Nov 25, 2026)
- Phase 2 (record tables and dashboard switch done and live Nov 25, 2026): New data model in Supabase (accounts, contacts, leads, quotes, policies, drivers, vehicles, properties, carriers, commissions, tasks, activities, documents) and migration of existing data
- Phase 3 (done and live Nov 26, 2026): Redesigned UI (Home, Client 360, Leads pipeline, Policy detail, Commissions, phone bottom bar)
- Phase 4 (in progress Nov 26, 2026): Leads intake, commission reconciliation, documents and activity log, helper login with limited access
- Phase 5: Automations (renewal and follow-up emails, reminders, scheduled reports)
