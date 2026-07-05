# Luckan Energy Staff Portal — Project Context

> Read this file fully before making any changes. It is the accumulated knowledge
> from months of development. Update it when you learn something new.

## What this is
Single-file web app (`index.html`, ~6,400 lines) for Luckan Energy (Qwest Consult PTY Ltd),
a licensed wholesale fuel & LPG supplier in Cape Town. Manages fuel/LPG/purchase orders,
debtors, clients, stock, assets, sales pipeline, documents, and sales reports.

- **Live:** https://luckanenergydash.netlify.app (Netlify Pro, auto-deploys from this repo, ~15s)
- **Owner:** Waleed Luckan (waleed@qwestconsult.co.za). Prefers Claude to act directly
  with tools, not give manual instructions. Communicate concisely.
- **Backend:** Supabase project `ujntmxeicffigfkwunxn` (Postgres + Auth + Storage + Edge Functions)
- **Backups:** private repo `LuckanEnergy/luckan-backups` — nightly 03:00 SAST via
  Edge Function `nightly-backup` + pg_cron (`0 1 * * *` UTC). Backs up all tables (JSON)
  and mirrors storage files (max 20 new/night).

## Architecture
- Everything in one `index.html`: CSS in `<style>`, one main `<script>` at the bottom.
- Data globals: FO (fuel orders), LO (lpg), PO (purchase), CL (clients), DB (debtors),
  AR (assets), DO/FL (documents/folders, localStorage only), LEADS, STOCK, CTR (id counter).
- Supabase REST via `dbGet/dbUpsert/dbDelete/dbPatch` helpers with `authToken()` auth.
- localStorage is a write-through cache; Supabase is the source of truth. `loadX(cb)`
  functions (callback pattern) pull fresh data; nest them on startup for render order.
- Pages are `pg-*` divs toggled by `goPage(page, el)`; render functions `rDash, rFuel,
  rLPG, rAll, rPO, rClients, rFuelDebtors/rLPGDebtors (via rDebtorsByType), rLeads,
  rRpt, rAssets, rDocs, rStockFuel/rStockLPG`.

## Auth & roles (live since 2026-07-02)
- Supabase Auth email+password. Users: Waleed=admin, Aashiqah Dramat=manager
  (aashiqah@qwestconsult.co.za), Saleigh Osman=staff (osmans@qwestconsult.co.za).
- `public.user_roles` table + `public.get_role()` (SECURITY DEFINER) used in all RLS.
- RLS: all tables require `authenticated`. Debtors: admin+manager only. DELETE: admin
  only, everywhere. Storage bucket `portal-docs`: authenticated CRUD, delete admin-only.
- Client code: `ACCESS_TOKEN` global set on login; `authToken()` returns it (falls back
  to anon key pre-login). ALL Supabase calls (REST + storage XHR) must use `authToken()`.
- Role UI gating: `ROLE_BLOCKED_PAGES`, `applyRoleUI()`, `pageAllowed()` in goPage,
  CSS `body[data-role=...] .btn-d{display:none}`.
- IMPORTANT: users were created via SQL insert into auth.users. If a new user gets
  "Database error querying schema" on login, run:
  `UPDATE auth.users SET confirmation_token=COALESCE(confirmation_token,''), ...` (all
  token columns COALESCE to '') for that email.

## Deploy workflow (critical — follow exactly)
1. Fetch: GET GitHub contents API for `index.html`, base64-decode to local file.
2. Edit with python string replacement (str_replace tools struggle with nested JS quotes).
3. **ALWAYS syntax-check before push:** extract the main `<script>` block, run
   `node -e "new Function(code)"`. On error, binary-search by splitting on `\nfunction `
   (ignore "Unexpected end of input" false positives).
4. Push: GET current file sha → PUT with base64 content + sha + commit message.
5. Bump `<meta name="app-version">` on significant pushes.
- Auth: fine-grained GitHub PAT scoped to luckan-portal + luckan-backups (Contents RW).
  Waleed supplies it; NEVER hardcode tokens in this repo or the portal code.
  GitHub rejects pushes containing secret strings (422).

## Recurring bug patterns (do not repeat these)
- **Missing `return` on saveX_supa:** every saveX_supa MUST return the dbUpsert promise,
  or Promise.resolve().then chains race ahead of the DB write. Root cause of multiple
  "delivered order didn't reach debtors/txlog" bugs.
- **Schema drift:** adding a field to a save payload without `ALTER TABLE ... ADD COLUMN
  IF NOT EXISTS` makes Supabase reject the whole upsert SILENTLY in old code. saveFO_supa
  now alerts on failure — keep that pattern for all saves.
- **IDs:** never `ARRAY.length+1` (collisions after refresh → upsert overwrites). Use
  `Date.now()`. `syncCTR()` keeps CTR ahead of max numeric id.
- **`confirm()` is blocked on mobile Samsung/Safari** — use `showConfirm(title,msg,label,onOk)`.
- **onclick strings:** use `&quot;` around ids inside generated HTML attributes, never
  bare `''+id+''` (breaks parsing).
- **Samsung Browser:** CSS variables in JS-generated HTML can render transparent —
  hardcode hex (#E8185A pink, #2abcd0 teal, #0d1b2a dark, #64748b muted, #e2e8f0 border,
  #f8f9fa light) inside modal/popup HTML. Unicode … ─ etc. in JS strings can break its
  parser — use ASCII or HTML entities.
- **Duplicate element IDs** have twice caused "renders to wrong empty element" bugs
  (duplicate page divs, even a full duplicate <html> document once). Check with
  `grep -oE 'id="[^"]+"' | sort | uniq -d` after structural edits.
- **pushOrderToDebtors preserves paymentStatus** on existing order rows — never reset
  paid→unpaid on sync.
- **Images:** `compressImage()` shrinks photos to ≤1600px JPEG before upload. Keep.

## Business rules
- Order flow: pending/processing (active table) → delivered → transaction log + debtor
  (auto-created/matched by client name/company, case-insensitive) → marked paid in
  Fuel/LPG Debtors → appears in Sales Report (revenue). Net profit = paid revenue −
  received PO costs.
- Debtors split: ⛽ Fuel Debtors & 🔥 LPG Debtors (order rows carry `type`).
- PO flow: pending (active) → received → PO transaction log + stock update.
- Fuel orders support two products on one order (f-add-prod2 checkbox).
- Delivered orders STAY in FO/LO with status='delivered' (txlog renders from them).

## Current state / pending
- Security done: auth+RLS, storage locked, nightly backups (tables+files), token rotated.
- 2026-07-05 hardening: portal-docs bucket is now PRIVATE. All file access (docs,
  order invoices, PO attachments) goes through short-lived signed URLs —
  `signedFileUrl()`/`openStoredFile()` helpers; uploads store the storage `path`;
  `viewDoc`/`viewInvoice` and the attachment buttons call them. Do NOT re-enable the
  public bucket or revert to STORAGE_PUBLIC links. activity (audit) log is append-only
  (UPDATE policy dropped). get_role() EXECUTE revoked from anon (authenticated only).
- Pending wishlist: enable leaked-password protection (Auth setting), audit log +
  soft deletes, change-password UI in Settings, MFA for admin, flip luckan-portal repo
  private, desktop side-by-side layouts, txlog card view on mobile.

## Working in Claude Code (local git workflow)
When running as Claude Code on a local clone, replace the GitHub-API deploy workflow with:
1. Edit `index.html` directly in the working tree.
2. **Syntax-check before every commit** (non-negotiable): extract the main `<script>`
   block and run it through `node -e "new Function(code)"` — same binary-search
   debugging approach on errors as described above.
3. `git add index.html && git commit -m "<clear message>" && git push origin main`.
   Netlify auto-deploys from main in ~15s.
4. Bump `<meta name="app-version">` on significant changes.
- Push auth: Waleed supplies a fine-grained PAT (Contents RW on this repo +
  luckan-backups). Configure once via git credential helper; NEVER commit it to
  any file in the repo.
- Supabase work (migrations, RLS, SQL) goes through the Supabase MCP connector if
  available, project id `ujntmxeicffigfkwunxn`. Never weaken RLS policies.
- Test pages after deploy on Samsung Internet specifically — it is the primary
  browser used by staff and has the strictest rendering quirks (see bug patterns).
