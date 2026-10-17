# MASTER PROMPT — Build a Customized ERP (blueprint derived from "FORESTA ERP")

> **How to use this file**
> 1. Open a new Claude Code session in an empty folder.
> 2. Paste everything below the line `=== BEGIN PROMPT ===` as your first message.
> 3. Either fill in the **PROJECT BRIEF** (Section 2) before pasting, or leave it as-is and the AI will interview you (Section 0) and fill it in itself.
> 4. The AI will answer with a plan and wait for your approval before building. Approve phase by phase.
>
> Language: instructions are English. User-facing UI labels stay **Indonesian** by default (kept exactly where quoted), switchable in the Brief.

=== BEGIN PROMPT ===

# 0. YOUR ROLE AND OPERATING PROTOCOL

You are a senior full-stack engineer and ERP business analyst. You will build a complete, production-grade, multi-user, role-based ERP web application for the company described in the **PROJECT BRIEF** (Section 2). The behavior to implement is distilled from a real, running ERP ("FORESTA ERP") for a heavy-equipment sales + spare-parts + workshop company in Indonesia. That ERP's modules, workflows and hard-won bug-fix lessons are the blueprint; the **company, industry, vocabulary, roles and enabled modules are replaced by the Brief**.

**Protocol — follow in this order, do not skip:**
1. **Intake.** If any Brief field below is still `{{...}}` or marked `ASK`, ask me the missing questions in ONE batched message (max 12 questions, multiple-choice with sensible defaults wherever possible). Do not ask about anything the Brief already answers or that this prompt gives a default for.
2. **Domain mapping.** Produce a table mapping each blueprint module/term to my industry's equivalent (Section 3), and list modules you propose to enable, rename, merge or drop. Wait for my OK.
3. **Plan.** Produce a phased build plan (Section 12) with the tech stack, folder layout, database schema outline and risks. Wait for my approval.
4. **Build phase by phase.** After each phase: run it, run the acceptance checks (Section 13), show what works, list what is not done, and wait for "continue".
5. **Never invent business rules silently.** If a rule is not specified here or in the Brief, choose the safest default, state it in one line, and add it to `docs/ASSUMPTIONS.md`.
6. Keep a `docs/` folder: `ASSUMPTIONS.md`, `DATA_MODEL.md`, `ROLES_MATRIX.md`, `MODULES.md`, `RUNBOOK.md` (deploy, backup, restore, add user, add module).

---

# 1. PRODUCT SUMMARY (what is being built)

A single web ERP that covers the full order-to-cash and procure-to-pay cycle, inventory, field/workshop operations, finance/accounting, HR/payroll, KPI/analytics and administration, for one company, with ~15–20 staff roles. Core characteristics that must be preserved:

- **Document-driven**: every business event is a numbered document (quotation → sales order → requisition → purchase order → goods receipt → delivery → invoice → payment), each linked to its predecessor so the whole chain is traceable.
- **Approval-driven**: owner/GM approval gates on money-affecting documents, with a rejection reason and full history.
- **Role-driven**: each role sees only its modules and only the rows it is entitled to; sensitive modules (payroll, per-person analytics) are hard-locked.
- **Audit-driven**: every create/edit/approve/reject/delete is logged with actor, role, time and detail.
- **Accounting-integrated**: business events auto-post balanced journal entries to a general ledger.
- **Mobile-usable**: staff use phones in the field; every screen must be usable at 375 px.

---

# 2. PROJECT BRIEF (fill in, or let the AI interview me)

```yaml
company:
  name: "{{COMPANY_NAME}}"              # ASK
  legal_form: "{{PT / CV / Ltd / ...}}"
  address: "{{ADDRESS}}"
  phone: "{{PHONE}}"
  email: "{{EMAIL}}"
  tax_id: "{{NPWP / VAT / EIN}}"
  logo: "{{optional upload}}"
  bank_accounts: ["{{BANK 1}}", "{{BANK 2}}"]   # used for invoice footer + bank reconciliation
industry:
  description: "{{ONE PARAGRAPH: what the company sells/does, how it earns money}}"   # ASK
  sells_physical_goods: true|false        # drives inventory modules
  sells_services_or_projects: true|false  # drives project/service modules
  has_workshop_or_field_team: true|false  # drives Mekanik/Field Service modules
  manufactures: true|false                # if true, add Production/BOM-to-WIP flow (Section 3)
  has_overseas_purchasing: true|false     # drives PO Overseas + multi-currency
localization:
  country: "ID"                           # drives tax, payroll law, date format, language
  ui_language: "id"                       # id | en  (blueprint labels are Indonesian)
  currency: "IDR"                         # base currency; others allowed per document
  vat_rate_percent: 11
  fiscal_year_start_month: 1
  timezone: "Asia/Jakarta"
document_numbering:
  pattern: "{TYPE}-{SEQ3}-{ORGCODE}-{ROMAN_MONTH}-{YEAR}"   # blueprint: SO-001-FRT-VII-2026
  org_code: "{{3-letter code}}"           # blueprint: FRT
  sequence_resets: never                  # blueprint: running counter NEVER resets; only suffix changes
roles_to_enable: ["owner","gm","finance_director","sales_manager","sales_admin","service_manager",
  "workshop_manager","inventory","purchasing","purchasing_overseas","storage","logistic","finance",
  "hrd","hrd_payroll","mekanik","freelance","driver"]     # rename/remove/add freely
modules_to_enable: ALL                    # or list ids from Section 7; or "ALL except: [...]"
term_overrides:                           # Foresta word -> my word (UI + docs)
  Mekanik: "{{e.g. Technician / Field Engineer}}"
  Unit: "{{e.g. Product / Machine / Package}}"
  Part: "{{e.g. Item / SKU / Material}}"
  Mill: "{{e.g. Site / Branch / Project Location}}"
  Client category list: ["{{SECTOR 1}}", "{{SECTOR 2}}", "..."]
approval_policy:
  approver_roles: ["owner","gm","finance_director"]       # called "isOwnerLike" in the blueprint
  invoice_requires_approval: false        # blueprint changed to auto-approved on 2026-09-25
  delete_requires_second_approval: true
commission_and_kpi:
  sales_commission: {enabled: true, new_customer_pct: 1.2, returning_customer_pct: 0.8}
  sales_kpi_targets: {revenue_per_month: "{{n}}", new_contacts: 15, followups: 30, quotations: 10}
data_migration:
  existing_data: "{{none | Excel files | other system}}"
deployment:
  target: "{{GitHub Pages / Netlify / Vercel / VPS}}"
  backend: "Supabase (Postgres + Auth + Row Level Security)"
```

---

# 3. DOMAIN MAPPING RULES (how to adapt the blueprint to any industry)

The blueprint is a **trading + service + workshop** company. Map as follows; propose the mapping to me in step 2 of the protocol.

| Blueprint concept | Generic meaning | Adapt by |
|---|---|---|
| Master Part | Stockable item catalog (SKU, name, description, category, sub-category, unit, stock, price, optional **BOM of sub-items**) | Rename to Item/Product/Material. Keep stock + BOM + categories. |
| Master Unit (BOM) | A sellable "package/assembly" made of catalog items (code, name, price, **bill of materials**, BOM revision stamp) | Rename to Product/Package/Machine. Drop if not needed. |
| Master Mill | Customer **site/location** database (name, company, address, province, regency) | Rename to Site/Branch/Project. Used by maintenance + field reports. |
| Database Client | Customers (category/sector, group, assigned sales owners, credit rating) | Keep. Replace category list. |
| Database Supplier | Local vs overseas suppliers | Keep; drop overseas if `has_overseas_purchasing=false`. |
| Unit Quotation / Parts & Service Quotation | Two quotation types (big-ticket product vs repeat parts/service) | Keep both or collapse into one with a `type` field. |
| Sales Order with fulfillment routing `IR` / `MR` / `BOTH` | Routes an order to **procurement** (IR), **field/workshop job** (MR), or both | Keep routing; rename MR (Mekanik Request) per industry. If no field work, hide MR. |
| Mekanik Report (visit, DAR, part recommendation, status report, BAPP, surat tugas, inspection) | **Field service / job execution documentation** | Rename; keep the visit → reports → completion chain. Drop if no field team. |
| Maintenance Schedule (3-month) | Recurring service reminders per sold asset | Keep if after-sales service exists; interval configurable. |
| Uang Dinas / Ticketing / Operation Cost | Field allowances, travel tickets, operational expenses | Keep as "Field Allowance / Travel / Opex". |
| Manufacturing (if `manufactures=true`) | NOT in the blueprint | **Add**: Work Order, material issue (reuse Parts Use), finished-goods receipt (reuse Goods Receipt pattern), BOM explosion (reuse), WIP/COGS journal posts. |
| Services-only company | No stock | Disable Master Part/Goods Receipt/DO/Mutasi; invoice directly from SO/project; keep everything else. |

Everything under **Finance, HR, KPI/Analytics, Trail, Settings** is industry-neutral and should be kept as-is (with localization from the Brief).

---

# 4. TECH STACK AND ARCHITECTURE (required unless I override in the Brief)

**Why this differs from the blueprint:** the blueprint stored ~55 collections as whole JSON arrays in one key/value table, rewrote the entire array on every save, and enforced permissions only in browser code. That produced real data-loss races (stock increments silently lost, documents overwritten, phantom "approved but not applied" records), 30-second freezes on a 2,900-row list, and security findings. **Do not copy that design. Copy the behavior, not the storage model.**

- **Frontend:** Vite + React 18 + TypeScript (strict), React Router, TanStack Query for server state, a small component library of your own (Table, Modal, Btn, Input, Select, Badge, Card, CollapsibleSection, PartPicker/ItemPicker, MobileCardList, DialogHost with `askConfirm`/`askPrompt`), CSS variables for light/dark themes. Virtualize any table > 200 rows.
- **Backend:** Supabase (Postgres, Auth, Realtime, Storage, Row Level Security). Real relational tables with foreign keys, one row per record, line items in child tables (or `jsonb` only for truly schema-less payloads such as inspection checklists).
- **Atomic server-side operations (Postgres functions/RPC) — mandatory for:** document numbering, stock increments/decrements, approval transitions, payment recording, journal posting, edit-request approval, stock reservation. The client must never read-modify-write a whole list.
- **Document numbers:** generated server-side by a function using a per-type counter row locked `FOR UPDATE` (no gaps requirement, but **no duplicates ever**). Pattern from the Brief. Revisions append `-R1`, `-R2`. Invoice numbers are **never renumbered** after issue (tax implications).
- **Concurrency:** optimistic UI is fine, but every mutation returns the authoritative row; failures surface a visible error (never `console.error` only). Use `updated_at` / row version checks on edits; a conflicting edit shows "this record changed, reload".
- **Auth:** Supabase Auth with username → synthetic email (`username@<company>.local`); `profiles` row holds name, username, role, `active`, `custom_modules text[]`, plus HR/payroll fields. **No default passwords like `username123`** (the blueprint had this; it is a security defect). Create users with a one-time random password and force change on first login. SHA-256 client-side hashing is NOT a substitute for real auth — don't do it.
- **Authorization:** enforced in **RLS policies and RPCs**, mirrored in the UI for UX only. Role/module rules from Section 5 must exist as SQL, not just React conditionals.
- **Realtime:** subscribe to changed tables for the visible module; debounce refetch. No full-store polling.
- **Security baseline (all were audit findings in the blueprint):** escape every user string interpolated into print/HTML; IDs from `crypto.randomUUID()` or DB sequences, never `Math.random()`; no secrets in the repo; CSP-friendly; rate-limit login (Supabase default) ; file uploads (logo, signatures) size-limited and type-checked.
- **Printing/PDF:** printable HTML templates per document (letterhead from Company Settings, signature images, bank info, terms), `window.print()` to PDF. Client-facing prints must hide internal fields (see 6.9).
- **Excel:** import/export with a styled-xlsx library (SheetJS fork with styles) — headers, bordered tables, `Rp` number format, totals, signature block with long-form date.
- **Offline-tolerant UX:** cache last-loaded read data; never cache writes.
- **Testing:** Vitest for pure business logic (stock allocation, totals, payroll, overtime, commission, numbering), Playwright smoke tests for the critical chain (Section 13), SQL tests for RLS (each role can/can't).
- **Deployment:** static frontend (GitHub Pages/Netlify/Vercel) + Supabase project. Provide `supabase/migrations/*.sql`, seed script, `.env.example`, and a README with step-by-step deploy. Frontend must send **no-cache** for `index.html` (the blueprint's stale-build bug).
- **Build identifier:** show `APP_BUILD` (date.sequence) in the footer; log it with errors.

---

# 5. ROLES, ACCESS AND PERMISSION MODEL

## 5.1 Mechanics (preserve exactly)
- `profiles.role` selects a default module list. `profiles.custom_modules` (optional array) **overrides** the role default per user.
- **Admin grant:** a pseudo-entry `"<module_id>:admin"` in `custom_modules` grants full/admin behavior inside that module (see everyone's rows, do admin actions) without changing the user's role.
- **Forced modules for every user regardless of role/custom list:** `cuti` (leave request), `uang_makan` (own meal allowance), `master_part` (catalog lookup), `document_trail` (cross-module progress view).
- **Hard locks** (cannot be unlocked by custom modules): Payroll → owner, gm, hrd_payroll only. User Analysis → owner, gm, hrd, hrd_payroll only. Uang Dinas → super users + the designated workshop leads + admin-grant, and every mekanik/freelance sees **only their own** rows (filter in RLS, not just UI).
- **"Owner-like" = owner, gm, finance_director.** They approve documents, see everything, and are exempt from the second delete gate (they *are* the approver). Rename in the Brief if your org differs.
- The user whose username is the GM's is normalized to role `gm` at login (blueprint quirk — generalize as "designated GM account").
- **Client visibility:** sales managers only see customers assigned to them (`salesManagerUsernames[]`); all other roles see all customers; owner-like see all. Sales can request access to a client (edit-request flow).
- **Sidebar badges:** each module shows a count of items awaiting *this user's* action (e.g., GM sees quotations "Menunggu Approval GM"; workshop manager sees field reports awaiting approval; finance sees overdue invoices/POs). Same counts appear on the Dashboard "modules you can access" panel.

## 5.2 Default role → module map (starting point; adjust to Brief)
| Role | Modules |
|---|---|
| owner, gm | ALL |
| finance_director | dashboard, invoice, customer, supplier, financial_dashboard, client_history, attendance, jurnal, bank_recon, laporan_keuangan, price_analysis, price_analysis_purchase, client_analysis, komisi |
| sales_manager | dashboard, quotation_unit, customer(own), client_history, sales_activity, kpi_sales, attendance, price_analysis, client_analysis, komisi(own) |
| service_manager | dashboard, quotation_ps, customer, part_request, mekanik_report, master_mill, kpi_mekanik, attendance |
| sales_admin | dashboard, sales_order, customer, ticketing, attendance, mekanik_report, workshop_ops, price_analysis, delivery_order |
| inventory | dashboard, item_requisition, master_part, master_unit, attendance, mutasi_barang |
| purchasing / purchasing_overseas | dashboard, po_local / po_overseas, supplier, part_request, attendance, mutasi_barang, price_analysis_purchase |
| storage | dashboard, goods_receipt, master_part, attendance, mutasi_barang |
| logistic | dashboard, goods_receipt (final workshop confirmation), parts_use, delivery_order, attendance, mutasi_barang |
| finance | dashboard, invoice, financial_dashboard, attendance, jurnal, bank_recon |
| hrd | dashboard, attendance, user_analysis |
| hrd_payroll | dashboard, payroll, attendance, user_analysis |
| mekanik / freelance | dashboard, mekanik_report, part_request, parts_use, master_mill, kpi_mekanik, attendance, uang_dinas |
| workshop_manager | dashboard, workshop_dashboard, maintenance_schedule, kpi_mekanik, sales_order, item_requisition, part_request, goods_receipt, parts_use, delivery_order, mekanik_report, master_mill, attendance, mutasi_barang |
| driver | (empty list + forced modules; configurable later) |

Also provide an **admin UI (Manajemen User)**: create user, set role, set custom modules (checkbox matrix incl. `:admin` toggles), activate/deactivate (deactivated users vanish from every picker), reset password, edit HR/payroll fields, and a **rename cascade** (see 6.1.4).

---

# 6. GLOBAL BEHAVIORS (apply to every module)

## 6.1 Cross-cutting rules
1. **Approval flow (generic):** `Draft → Menunggu Approval GM → Disetujui GM | Ditolak GM (+ reason)`. Approver records `approvedBy`, `approvedDate`. Some modules use other approver labels (Workshop Manager "Septi", Service Manager "Asil", HRD) — make the approver role a per-module setting.
2. **Edit requests (maker–checker) for master data:** non-owner-like users cannot directly Add/Edit/Delete Master Part and Database Client rows; they submit an `edit_request {module, action: Tambah|Edit|Hapus, targetId, targetLabel, payload, requestedBy, status}`. Approver sees the diff; approval applies the change atomically (RPC) and logs it. Pending requests are never pruned.
3. **Universal two-gate delete:** gate 1 = confirm dialog ("Yakin untuk hapus item ini?"); gate 2 = for non-owner-like users the delete becomes a pending `Hapus` edit-request that applies only when an approver approves it. Owner-like users skip gate 2. Deleting a document must cascade/clean linked rows consistently (e.g., deleting a leave form removes its Absensi entries) — implement as RPC, not client loops. Prefer soft-delete with `deleted_at` for financial documents.
4. **Rename cascade:** because the blueprint snapshotted person names into documents, renaming a user cascaded exact-match string replacement across ~30 collections. In the new design store `user_id` FKs and display names via join, so a rename is automatic; keep a `*_name_snapshot` only for printed/legal documents, and provide the cascade for snapshots.
5. **Audit log:** `audit_log(id, ts, module, doc_nomor, action, detail, actor_name, actor_role)`; written by DB triggers/RPCs, not by UI. Actions include `Ajukan Tambah/Edit/Hapus`, `Approve`, `Reject`, `Edit (Superuser)`, `Simpan Draft`, payments, etc. Audit Trail module = filterable viewer (module, doc, actor, date), export to Excel. Keep 180 days by default (configurable); never prune pending items.
6. **User activity log:** login + module-navigation events per user (for HR/Owner "User Analysis" adoption stats). Same retention.
7. **Revisions:** a quotation revision gets a new number `<base>-R<n>`, links `revisiDari` to the base. A **revision family** helper must be the single source of truth for "does this quotation (any revision) already have an SO": forward lookups resolve through the whole family; reverse lookups (SO → source quotation) are exact-match. (The blueprint hit this bug three times before consolidating it.)
8. **Downstream flagging on superuser edit:** when an owner-like user edits an already-approved quotation that spawned an SO, flag every downstream SO/IR/PO for manual review (badge) and only auto-append a note to the SO if the superuser typed one and the SO isn't locked (no shipment / supplier PO / paid invoice). Never silently change qty/price downstream.
9. **BOM revision warning:** documents snapshot `{unitCode: bomUpdatedAt}` at creation; show a non-blocking "BOM revised since" badge on SO/IR if the live BOM stamp differs. Never auto-mutate documents.
10. **Client-facing documents must never show internal Part Description:** enforced on Quotation (both), Sales Order, Invoice, Delivery Order — on screen and in print. Also never print the internal "Sales Discount" kickback (see 7.3).
11. **Dates:** store ISO `YYYY-MM-DD`; display as `DD-MON-YY` (e.g., `05-OCT-26`) in tables; long Indonesian form (`12 Agustus 2026`) in report signature blocks.
12. **Money:** integer minor units or `numeric(18,2)`; format with thousands separators; multi-currency documents show `CUR amount`; `fmtMoney(n, currency)`.
13. **Payment terms:** `COD` or `TEMPO` (displayed on POs as "CREDIT"). TEMPO has a due date entered as a date **or** N days from today. `getDueStatus` → `COD | Belum Jatuh Tempo | Jatuh Tempo Segera (≤3 days) | Terlambat`.
14. **Uploads:** logo, per-user signature image (draw-on-canvas SignatureModal), used on prints.
15. **Light/dark theme** toggle persisted per browser; sidebar group collapse state persisted per browser; mobile: tables become card lists, grids collapse to 1 column, inputs ≥16 px, buttons ≥40 px tap targets.
16. **Dialogs:** never use browser `alert/confirm/prompt` for flows that need to await — implement promise-based `askConfirm(message,{danger,confirmLabel})` and `askPrompt(message)` via a DialogHost.
17. **Empty/loading/error states** for every list and form; never fail silently.

## 6.2 Sidebar structure
Standalone top items: `dashboard`, `financial_dashboard`, `workshop_dashboard`. Collapsible groups (Max/Min), in this order:
`TRAIL` (audit_trail, document_trail) · `DATABASE` (master_part, master_unit, master_mill, customer, supplier) · `SALES` (quotation_unit, quotation_ps, sales_activity, client_history, price_analysis, client_analysis, komisi) · `OPERATION` (sales_order, item_requisition, po_local, po_overseas, price_analysis_purchase) · `LOGISTIK` (goods_receipt, parts_use, delivery_order, mutasi_barang) · `WORKSHOP` (mekanik_report, maintenance_schedule, uang_dinas) · `EXPENSES` (ticketing, workshop_ops) · `FINANCE` (invoice, jurnal, bank_recon, laporan_keuangan) · `HR` (attendance, cuti, uang_makan, payroll, user_analysis) · `KPI` (kpi_mekanik, kpi_sales) · `SETTING` (users, company_settings). Any module not listed falls through as standalone so nothing silently disappears.

---

# 7. MODULE CATALOG (≈50 modules — implement all enabled in the Brief)

Notation: **[CORE]** keep in every industry · **[SWAP]** rename/adapt per industry · **[OPT]** only if the Brief enables it. Status strings are Indonesian and must be kept exactly (they are UI vocabulary). "Who" = role that normally acts.

## 7.1 Dashboards
- **dashboard [CORE]** — greeting + role badge; "modules you can access" with approval badges; due-date reminder panels: invoices (receivables) and POs (payables) coming due/overdue (finance-type roles only); recent activity.
- **financial_dashboard [CORE]** — receivables aging, payables aging, overdue invoices (non-"longOverdue") and overdue POs, cash in/out summaries, per-bank balance. Roles: owner, gm, finance, finance_director.
- **workshop_dashboard [OPT]** — pending field reports awaiting workshop-manager approval, jobs in progress, maintenance due.

## 7.2 Master data / Database
**master_part [SWAP]** — Item catalog. Fields: part number (unique, auto-generated next number on "Tambah" approval — collision-safe), name, description (internal), kategori (`Parts` | `Service`), sub-category (managed list, add/rename/delete; seed: ATK, Consumable, Electrical, Material, Hydraulic), stock (`stok`), unit price/cost, optional `bom[]` of component items (recursive explosion with cycle guard). Rules: non-owner-like edits go through edit-requests; list must handle ~3,000 rows (virtualized, server-side search by number/name/description); shows **available stock = stock − FIFO-reserved** (see 8.1); Excel import/export; stock revision action (approver) that writes an audit entry; status `Disetujui GM - Masuk Master Part` for items entering from Request Part Baru.
**master_unit [SWAP/OPT]** — "Unit (BOM)": code, name, price, `bom[] {partNumber, partName, qty}`, `bomUpdatedAt` stamp bumped on any BOM change (feeds the 6.1.9 warning).
**master_mill [OPT]** — site database: Mill Name, Company, Address, Province, Regency. Used as picker by field reports and maintenance. Writes via compare-and-swap in the blueprint → in new design normal row writes.
**customer [CORE]** — Database Client: name, category (fixed list from Brief; blueprint: SAWIT, MINING, MARINE, MANUFACTURE, GOVERNMENT, KONTRAKTOR, OIL & GAS, UTILITAS/TELCO, LOGISTIC, LOGGING, KAROSERI, DEALER, UMUM — doubles as "Sector" for Sales KPI), **group** (Master Group list: several companies under one holding), contacts, address, tax id, credit/payment terms, `salesManagerUsernames[]`, **payment-punctuality rating badge** (computed: % of settled TEMPO invoices paid on/before due date → `Sangat Baik ≥90 / Baik ≥70 / Cukup ≥50 / Kurang <50 / Belum Ada Data`, plus count of still-overdue invoices; shown in the client record AND on both quotation forms). Edit-requests for non-owner-like; access-request flow for sales managers; import/export Excel; "ClientFixCard" admin tool to repair legacy assignment data.
**supplier [CORE]** — local vs overseas lists (separate tabs), name, contacts, bank, terms, currency; Excel import.

## 7.3 Sales
**quotation_unit [SWAP]** — Unit Quotation (big-ticket). Who: sales_manager creates; owner-like approves. Statuses: `Draft` → `Menunggu Approval GM` → `Disetujui GM` | `Ditolak GM`; then `Sudah jadi SO`; plus follow-up/`ON GOING` views (stale >N days flagged; `followUpExtendedAt`). Fields: customer, items (unit or part lines: code, name, qty, price, subtotal), `clientDiskon` (client-visible discount, % or Rp), **internal `salesDiscount {aktif, nominal}` kickback (never shown to client, never leaks to SO/print)**, PPN % (default 11), `termOfPayment[] {label, percent|nominal, inputMode}` (e.g., DP1 30%, Final 70%; total ≠ 100% is allowed but shows a yellow badge, 100% green), delivery/lead time + PPN-inclusion boilerplate defaults, validity, notes, `revisiDari`. Actions: save draft, submit, edit in place (draft/under approval), **revise** (new `-R#`, cascades flags downstream), reassign sales manager, approve/reject(with reason), "Buat SO" (hands off to Sales Order with the quotation pre-selected), print. Auto-logs a Sales Activity entry on creation/submit (sales role only). Access check: non-owner-like can only quote customers assigned to them.
**quotation_ps [SWAP]** — Parts & Service Quotation: same engine, plus can **pull mechanic Part Recommendations** from field reports (`status: Ditarik ke Quotation`), approver label may be Service Manager then GM; `Menunggu` part-recommendation inbox with badge.
**sales_activity [CORE]** — daily log: visits, calls, operational costs (`biaya[]`, `totalBiaya`), customer, notes; entries auto-created (flag `auto:true`) on quotation/new-client events; feeds KPI Sales.
**client_history [CORE]** — read-only 360° per client: quotations, SOs, invoices, payments, activities.
**price_analysis [CORE]** — selling-price trend/variance per item/client/period (charts + table + export).
**client_analysis [CORE]** — client ranking by revenue, frequency, sector, new vs returning, concentration.
**komisi [OPT]** — Sales Commission (standalone from Payroll's own commission): per SO row. `commission base = (SO total − internal salesDiscount)` × rate, rate = **NEW 1.2% / RETURNING 0.8%**. NEW/RETURNING is **derived, not stored**: the customer's earliest SO (by date, tiebreak by number) is NEW, every later SO RETURNING. Eligibility = every SO with status `Aktif`. Owner-like can override salesDiscount and rate per SO and mark paid (`dibayar`, `dibayarTanggal`, `dibayarOleh`). Reps see only own rows and never the raw sales-discount amount. (Payroll keeps its own simpler `(subtotal−diskon)×rate` — retain both, documented.)
**kpi_sales [OPT]** — 4-pillar scorecard per salesperson: Pillar 1 Prospecting (new contacts), Pillar 2 Pipeline development (follow-ups, quotations), Pillar 3 Conversion & revenue (SO value vs target), Pillar 4 diversification (sectors approached/covered). Targets editable per person (defaults in Brief) and prorated by selected period; fixed "Target SO" blocks (this month vs monthly target, YTD vs 12× target). Roster override: add/exclude people without changing their login role.

## 7.4 Operation (order → procurement)
**sales_order [CORE]** — Created by sales_admin from an **approved** quotation by picking items (partial SOs allowed). SO fields: number, source quotation `sourceNomor`, client PO number (`noPoClient`), customer, items (carry price; only the client-visible discount carries over), subtotal, diskon (pro-rated ratio), pajak (same % as quotation), total, `termOfPayment` (inherited; if the quotation had none, the SO builder may define one — must sum to 100% here), shipment info, `fulfillmentType`, `status: Aktif` (lives forever as Aktif), `tipe` (Unit | Parts/Service). **Fulfillment routing:** `IR` (appears in Item Requisition shortfall picker), `MR` (skips IR, appears in Field Report "MR On Going" picker, auto-creates a project-level Uang Dinas placeholder), `BOTH` (**automatic for every Unit SO**). "Ubah Tipe" cycles IR→MR→BOTH→IR. Edit SO (with lock rules), mark selesai, print, delete via two-gate.
**item_requisition [CORE]** — "IR": created **only when stock is insufficient** for an SO (FIFO shortfall, 8.1) or manually; **no prices shown**; items `{partNumber, qtyButuh, ..., sudahDiPO}`; statuses `Menunggu Purchasing` → `Sudah di-PO` (when every item allocated to some PO); item-level qty edit; shortage computed from exploded BOM of the SO (Unit lines → BOM components × qty; Service lines skipped).
**po_local / po_overseas [CORE/OPT]** — one shared component. Pulls items from one or more IRs (marks consumed items `sudahDiPO`); supplier; currency + `kursReferensi` for overseas; `biayaTambahan[]` (freight, duties…); optional PPN 11% on items subtotal; `tipePembayaran` COD/TEMPO + due date; urgent flag; revise; import-cost record; supplier doc & invoice/faktur numbers. Statuses: `Menunggu Approval GM` → `Disetujui GM - Kirim ke Supplier` | `Ditolak GM`; receiving updates: `Diterima Sebagian` → `Diterima HO - Menunggu Konfirmasi Workshop` → `Sudah Diterima`; payment: `statusBayar` `Belum Dibayar` → `Sudah Dibayar` with `jumlahDibayar`, `tanggalBayar`, method (BCA/MANDIRI/CASH). Approved POs auto-create a Jurnal "Pengeluaran" entry. Creation, approval, rejection, payment and edits are all audit-logged (blueprint bug: creation was once not logged).
**price_analysis_purchase [CORE]** — purchase-price trend per item/supplier/currency.

## 7.5 Logistics / Inventory
**goods_receipt (Penerimaan Barang) [CORE]** — records receipt against a PO. Two modes: **(a) received directly at Workshop** → creates GR `statusFinal: "Final - Diterima Workshop"`, increments stock immediately, updates PO `receivedMap` + status; **(b) received at Head Office** → no stock change, PO status `Diterima HO - Menunggu Konfirmasi Workshop`, auto-creates an **Internal DO (Office → Workshop)**; stock only increments when Workshop/Logistic clicks "Konfirmasi Diterima Workshop". Per-item `qtyDiterima`, mekanik/receiver name, partial receipts accumulate in `receivedMap`. Authorized receivers = a GM-editable username list. GR edits go through an edit-request (`Menunggu Approval Workshop`). Stock increments must be **atomic RPC** (blueprint lost stock increments under concurrency).
**parts_use [OPT]** — Release of items from an SO to a mechanic: `qtyReleased` per item; **release does NOT change stock** (reservation just converts); return flow (`submitReturnRequest` → `Menunggu Konfirmasi Return` → confirmed, which increments stock); non-SO release; edit release. Updates the FIFO reservation (released qty no longer counts as reserved).
**delivery_order [CORE]** — Manual DO for an SO (Unit or Parts/Service); no description shown. Pick items with `shipQty ≤ sisa` (remaining unshipped); **block the whole DO if real stock < shipQty** (no silent auto-reduction); partial DO flag; status `Dikirim`; **decrements stock atomically**; edit DO; printable; Unit DO triggers Maintenance Schedule creation.
**mutasi_barang [CORE]** — read-only, fully derived stock-movement ledger: Masuk (from Goods Receipt) and Keluar (from Delivery Order, Parts Use returns, adjustments). Searchable by part number/name/description/customer name/customer PO/supplier PO. Never manually entered.

## 7.6 Workshop / Field service [OPT]
**mekanik_report [SWAP]** — per **Visit** (kunjungan: SO link or MR, customer/mill, assigned mekanik(s), dates, status incl. `Ditugaskan` → … → `Selesai`) three separate forms: **DAR** (Daily Activity Report; `status: Menunggu Approval Septi` → `Disetujui Septi` | `Ditolak Septi`), **Part Recommendation** (items+qty+reason; pulled into SQ), **Status Report**. Plus **BAPP** (Berita Acara Penyelesaian Pekerjaan: work type Instalasi/Comissioning/Service/Training/Inspeksi/…, day, unit condition `Berfungsi Dengan Normal/Break Down/Other`, signatures), **Surat Perintah Tugas**, **Inspection Report** (component checklists with conditions `NORMAL/DAMAGE/MISSING/LEAKING/CHANGE/OTHER`, hydraulic function checks, separate checklist for Truck Crane vs standard crane — make checklist templates data-driven so any industry can define its own). Severity options `Tidak ada Kendala/Ringan/Sedang/Berat/Kritikal`; work status `Selesai/Belum Selesai/Perlu Lanjutan/Menunggu Part`; unit condition `Working/Need Repair/Break Down`; berita-acara types `Pemasangan/Pemasangan & Commissioning/Training/Survey/Perbaikan…`. Marking a visit `Selesai` auto-creates **Uang Dinas** records per mekanik. "MR On Going" picker lists MR/BOTH SOs with no visit yet.
**part_request [OPT, currently disabled in blueprint]** — "Request Part Baru": mekanik → service manager review (`Menunggu Review Asil`) → GM approval (`Menunggu Approval GM`) → enters Master Part (`Disetujui GM - Masuk Master Part`). Keep code path but ship it **off by default** (it was deactivated 2026-07-13; keep the flag).
**maintenance_schedule [OPT]** — auto-generated per sold unit with **3-month interval** from DO/last service; fields `nextDueDate`, status `Terjadwal`/`Selesai`; due label `Terlambat` (<0 d) / `Segera` (≤14 d) / `Terjadwal`; workshop manager assigns mekanik; "Buat SQ" button when a client's last SO is empty/stale >3 months pre-opens Parts & Service Quotation with the client; client-name link jumps to Database Client. Badge counts Terlambat+Segera.
**uang_dinas [OPT]** — per-mekanik-per-trip allowance: auto-created when a visit is `Selesai` (and as project-level placeholder for MR SOs); amount = days × current payroll `uangDinasRate` (live rate, not the frozen row rate) + `uangSaku`; super users (owner-like + `uang_dinas:admin`) and designated workshop leads see/edit/pay everyone; mekanik sees only own rows; payment feeds Jurnal.
**ticketing [OPT]** — travel tickets (plane/ship/train-bus) for staff going to remote sites; sales_admin inputs; **no approval**; links SOs; posts to Jurnal as cost.
**workshop_ops [OPT]** — Operation Cost: operational expenses entered by sales_admin, categorized from the flat "Master COA" list (expense tagging), Excel export.

## 7.7 Finance
**invoice [CORE]** — Created by finance from an SO. **Three billing modes:** (1) **Termin** — SO has Term of Payment: pick a termin (DP/Pelunasan); invoice amount from `computeTerminAmount`; item rows printed at full "harga pokok", the percentage applies only to the subtotal. (2) **Per-item** — SO without termin: pick un-invoiced items. (3) **Ad-hoc DP** (legacy SOs without termin): DP as %/Rp/"sisa"(remaining), capped to remaining pre-tax base; once an SO has an ad-hoc DP invoice it stays in DP mode. Then: invoice-level discount (%/Rp) taken first, **then PPN 11% on the discounted amount** (ad-hoc DP bills at the SO's own tax ratio). Fields: soNomor, `noPoClient`, tipe, COD/TEMPO + due date (date or N days), `noFakturPajak`, `terbayar`, `statusBayar` (`Belum Lunas` → `Lunas`), `riwayatPembayaran[]` (each payment: date, amount, method BCA/MANDIRI/CASH, optional PPh withholding checkbox, admin fee), `longOverdue` flag (excluded from overdue badge). Policy: auto-approved on creation (`statusApproval: "Disetujui GM"`) by default; if Brief sets `invoice_requires_approval: true` use `Menunggu Approval GM` with edit requests (`Menunggu Approval GM` for edits). **On creation (or approval) auto-post revenue:** `Dr Piutang Usaha = total / Cr Pendapatan (Unit vs Jasa by SO tipe) = total − PPN / Cr PPN Keluaran = PPN`. **On each payment auto-post cash clearing** as ONE cumulative entry replacing the previous one (idempotent by source). Payments feed Jurnal "Pemasukan" and Bank Recon. Print with bank info from Company Settings. Invoice numbers are never migrated/renamed.
**jurnal [CORE]** — operational cash journal: `tipe` Pengeluaran / Pemasukan. Pengeluaran entries arrive automatically (approved POs, Payroll run, Uang Dinas, Ticketing, Workshop Ops) and manually; Pemasukan arrive from Invoice payments. Approve = **"Tandai Lunas"** via BCA / MANDIRI / CASH; for BCA/MANDIRI this writes a matching entry into Bank Recon with `sourceModule/sourceId` (the only place that writes auto bank entries). Edit/plain-edit rules for paid entries.
**bank_recon [CORE]** — per-bank (BCA, MANDIRI from Brief) statement ledger: entries `Masuk`/`Keluar`; auto-posted from Jurnal Lunas, plus manual (deposits, transfers, bank fees). **Saldo Awal is just a normal manual `Masuk` entry** (one flat list, no special schema). Running balance, reconcile-tick, Excel export.
**laporan_keuangan (LK) [CORE]** — real accounting, 3 phases built together:
- **Phase 1:** Chart of Accounts `{kode, nama, tipe (Aset/Liabilitas/Ekuitas/Pendapatan/Beban), subTipe, saldoNormal}` (seed from Section 9), **Saldo Awal** (opening balances), manual **Jurnal Umum** (multi-line, balanced debit=credit), General Ledger, Trial Balance, Laba Rugi (P&L), Neraca (Balance Sheet), Arus Kas if feasible. Kept **separate** from the flat expense-tag "Master COA" (7.8).
- **Phase 2 — Account Mapping + Auto-posting:** UI "Pemetaan Akun" maps semantic roles (`piutangUsaha, pendapatanUnit, pendapatanJasa, ppnKeluaran, ppnMasukan, hutangUsahaLokal, hutangUsahaOverseas, persediaan, kas, bankBCA, bankMandiri, hutangGaji, bebanGaji, bebanLembur, hutangKomisi, pph21, …`) to COA codes — **never hardcode codes**. `lkAutoPost(sourceType, sourceId, date, narration, lines[])` rules: drop zero-lines; need ≥2 non-zero lines; every kode must exist; **|Σdebit−Σkredit| ≤ 1 else do nothing**; **upsert by (sourceType, sourceId)** so re-fired events replace instead of duplicating; flag `auto:true`, `createdBy:"SYSTEM"`; entries remain editable/deletable; **must never throw or block the business action** (no-ops silently if mapping is missing, and shows an "unposted events" warning list in LK). `lkAutoUnpost(sourceType, sourceId)` for reversals. Auto-post sources: invoice (revenue), invoice payment (cash), PO approval/receipt (inventory/AP), PO payment, payroll run, uang dinas, commission, opex.
- **Phase 3 — Fixed Assets:** register with own COA triple (aset / akumulasi penyusutan / beban penyusutan) per asset; straight-line depreciation auto-posted monthly **only from the date the asset was entered into the system forward** (earlier depreciation is assumed inside Saldo Awal — no double count); disposal flow.
- Roles: owner, gm, finance, finance_director.

## 7.8 HR
**attendance (Absensi) [CORE]** — (a) daily check-in records (Hadir, Cuti, Ijin, Sakit, Dinas Luar…), (b) **"Rekap Mesin"**: import a fingerprint-machine weekly XLS, parse per-employee totals (normal days, riil, cuti, ijin, sakit, lembur hours…), then **HRD correction layer** (`hitungKoreksiAbsensi`): fields `normalOverride, riilOverride, dLuar, cuti, cutiKhusus, jmlIjinOverride, ijin, setengahHari, sakit, absenManual, lemburLibur`; rules: `Riil` = on-site only (Dinas Luar is its **own** bucket, never merged into Riil); `Absen = max(0, normal − riil − dLuar − otherLeave)` unless `absenManual`; `otherLeave = cuti + cutiKhusus + jmlIjin + ijin + setengahHari + sakit` (a half-day counts as a full *accounted* day for Absen/Uang Kerajinan but ×0.5 toward annual Ijin quota and 0.5× toward Uang Makan); `normal = k.normal − hariLibur` unless full-dinas-luar; (c) weekly report (Laporan Mingguan) with a manageable mekanik roster; (d) dashboard panel; (e) Excel import (multi-sheet) and export. Saved recaps feed Payroll.
**cuti (Form Cuti) [CORE, visible to every user]** — leave request → `Menunggu Approval HRD` → approved by HRD/owner/gm (`hasPayrollAccess`); quota **12 days/year**; approval writes `Cuti` entries into Absensi; leave beyond quota deducts salary `gajiPokok / hariKerjaBulanIni × cutiKelebihan`; HRD can file on behalf of staff; owner/GM can delete (cleanup of Absensi entries cascades). Number format `CT-…` stored in `id`.
**uang_makan [CORE, visible to every user]** — own monthly meal allowance report; `Uang Makan = Σ riilHari × uangHarianRate` (+ 0.5× for half-days); read-only; owner/gm/hrd_payroll see everyone.
**payroll [CORE, locked]** — owner/gm/hrd_payroll only. Tabs: **Proses Payroll** (monthly run), **Gaji & BPJS** (master pay data, inline-edit cells, status Tetap/Harian/Freelance toggle, bulk Excel export→edit→import keyed by `username`), **Data Karyawan** (ID/KK/family). Per-employee: `gajiPokok, tunjangan, bpjsTK%, bpjsKes%, uangHarianRate, uangDinasRate, uangKerajinan, jabatan, statusKaryawan, signature…`. **Net pay = gajiPokok + tunjangan + lembur + komisi + uangMakan (+ kerajinan) − potonganManual − potonganCuti − BPJS TK − BPJS Kesehatan (− PPh21 if enabled)**. Lembur per Indonesian PP 35/2021 (see 9.2). Commission line = payroll's simpler `calcKomisi`. Run produces payslips (printable "slip gaji", escaped HTML), saves `payroll_runs`, posts to Jurnal/LK (hutang gaji, beban gaji, lembur, BPJS, PPh21), Excel export with letterhead.
**user_analysis [CORE, locked]** — per-person login/module-usage analytics from the activity log (adoption heat-map, last login, modules visited, inactivity).

## 7.9 KPI / Analytics
**kpi_mekanik [OPT]** — field staff scorecard from reports: jobs, on-time completion, severity mix, visits, part-recommendation conversion; period filter; export.
**kpi_sales** — see 7.3.
**document_trail [CORE, visible to all]** — cross-module progress view for a chain **SO → IR → PO → GR → DO (→ Invoice)**: given an SO/PO/customer, show each stage's document, status, qty fulfilled, and what's blocking. Exists so every department can check job status without opening each module.
**audit_trail [CORE]** — see 6.1.5.

## 7.10 Settings / Admin
**users [CORE]** — see 5.2 (create/edit/deactivate, roles, custom modules incl. `:admin`, HR fields, password reset, rename cascade).
**company_settings [CORE]** — name, address, phone, email, tax id, logo, bank name/holder/account (used on invoice print — **editable, not hardcoded**), configurable lists (client categories, part sub-categories, GR receiver usernames, KPI roster, mekanik roster), `APP_BUILD`, and the **Legacy Document-Number Migration tool**: read-only preview then apply a one-time rename of old-format numbers to the unified format with **every cross-reference rewritten in the same pass** (SO.sourceNomor, IR/DO/Invoice/PartsUse/Visit.soNomor, PO.irNomors, GR.poNomor, InternalDO.grNomor, Ticket.soNomors, quotation `revisiDari` chain incl. `-R#` children, audit-log docNomor); **Invoice numbers excluded**. Include only if I import legacy data.
**master COA (flat) [OPT]** — office operational cost accounts used to tag Opex/Biaya Lain/Jurnal exchange-rate differences; separate from LK COA.

---

# 8. CORE ALGORITHMS (implement as unit-tested pure functions + SQL where atomic)

## 8.1 Stock, reservation and shortfall (FIFO allocation)
- `stok` rises on **final** Goods Receipt (Workshop), on Parts Use return confirmation, on approved stock revision; falls on Delivery Order. **IR creation, PO creation, PO receipt at HO, and Parts Use release do NOT change `stok`.**
- **Explosion:** `explodeSOItems(so)` → `{partNumber: qtyNeeded}`. Unit lines → every BOM component × qty; a part with its own `bom` is exploded recursively (cycle guard via `seen` set); `kategori === "Service"` lines are skipped; merge duplicates.
- **FIFO allocation:** for each part, take every SO with `status === "Aktif"`, compute `need = explodedNeed − qtyReleasedViaPartsUse`, sort by SO date then number, hand out physical `stok` in that order → `{soNomor: {allocated, shortfall}}`. This guarantees Σshortfall = max(0, Σneed − stok) (naive "sum everyone else's need" double-counts).
- `reservedQtyFor(part, excludeSo)` = Σallocated of all other SOs. Available = stok − reserved. IR shortfall = that SO's `shortfall`.
- **Bulk variant:** compute allocation for ALL parts in a single pass (explode each active SO once). The per-part version froze the UI for 30+ s at ~3,000 parts. In the new design do this in SQL (materialized view / function) and cache it.

## 8.2 Document numbering
`TYPE-SEQ3-ORG-ROMANMONTH-YEAR[-R#]`. `SEQ` = (max sequence ever used for this TYPE across all months/years) + 1 — **does not reset**. Type codes from blueprint: `UQ` unit quotation, `SQ` parts/service quotation, `SO`, `IR`, `PL` PO lokal, `PO` PO overseas, `GR`, `DO`, `INV`(never migrated), `CT` cuti, `IDO` internal DO, etc. (confirm full list from the Brief). Server-side counter, locked row, unique constraint on `(type, nomor)`.

## 8.3 Totals
`subtotal = Σ qty×harga`; client discount % or Rp; `taxBase = subtotal − diskon`; `pajak = taxBase × pct/100`; `total = taxBase + pajak`; PO: `+ Σ biayaTambahan`, PPN only on items subtotal. SO pro-rates quotation discount/tax by `selectedSubtotal/quotationSubtotal`. Termin amount = percent × (SO pre-tax) or fixed Rp; invoice then applies its own discount then PPN.

## 8.4 Payment status / aging
`terbayar` accumulates; `Lunas` when `terbayar + pph + adminFee ≥ total`; due-status from 6.1.13; client rating from 7.2; PO `statusBayar` likewise.

## 8.5 Journal auto-post contract
See 7.7 (LK Phase 2). Pure function `buildLines(event, mapping) → lines[] | null`; unit-test balance and idempotency.

## 8.6 Overtime (Indonesia, PP 35/2021 Pasal 31–32) — localization-dependent
hourly = monthly wage / 173. Working day: hour 1 = 1.5×, further hours 2×. Rest/holiday, 6-day week: hrs 1–7 = 2×, hr 8 = 3×, hrs 9–11 = 4×. 5-day week: hrs 1–8 = 2×, hr 9 = 3×, hrs 10–12 = 4×. Cap 4 h/day, 18 h/week (warn when weekly > 18). Wage base = gajiPokok + tunjangan tetap. **If `country` ≠ ID, replace with the local rule set from the Brief and say so in ASSUMPTIONS.md.**

---

# 9. SEED DATA AND REFERENCE LISTS

9.1 **Chart of Accounts (LK) seed** — numeric, grouped: 1xxx Aset (1101 Kas Kecil, 1102 Kas Besar, 1111 Bank BCA, 1112 Bank Mandiri, 1113 Bank Lainnya, 1120 Piutang Usaha, 1121 Cadangan Kerugian Piutang, 1130 Piutang Karyawan, 1140 Persediaan Spare Parts, 1141 Persediaan Barang Dalam Perjalanan, 1150 PPN Masukan, 1160 Uang Muka Pembelian, 1170 Biaya Dibayar Dimuka, fixed assets + akumulasi), 2xxx Liabilitas (2110/2111 Hutang Usaha Lokal/Overseas, 2120 Uang Muka Penjualan, 2130 Hutang Gaji, 2131 Hutang Komisi, 2132 Hutang Biaya Lain, 2140 PPN Keluaran, 2141 PPh 21, 2142 PPh 23, 2143 PPh Final, 2150 Hutang Bank, 2160 Biaya Masih Harus Dibayar), 3xxx Ekuitas, 4xxx Pendapatan (Pendapatan Unit, Pendapatan Jasa/Service, Parts), 5xxx Beban (5210 Gaji Pokok, 5211 Lembur, tunjangan, BPJS, komisi, penyusutan per kategori aset, opex). Generate a complete, balanced-by-type list; **make it editable**. Rename inventory/revenue accounts to the Brief's industry.
9.2 **Other lists:** client categories (Brief), part sub-categories, inspection checklists (data-driven templates), severity/status/condition options (7.6), `HARI_OPTIONS` Senin–Minggu, Indonesian months.
9.3 **Users:** seed only an `owner` admin with a random one-time password printed once at setup; no demo accounts in production seeds. Provide a separate `seed:demo` script that creates one user per role + realistic sample customers/items/documents for demos and Playwright tests.
9.4 **Settings defaults:** company info from Brief; bank info; thresholds (due-soon 3 days, maintenance interval 3 months, maintenance "Segera" 14 days, leave quota 12, audit retention 180 days).

---

# 10. IMPORT / EXPORT / PRINT INVENTORY

- Excel import: Master Part, Customers, Suppliers, Payroll master (Gaji & BPJS, keyed by `username`), fingerprint Rekap Mesin (multi-sheet), Absensi. Validate headers with accepted aliases, show a preview + error rows, never partially apply silently.
- Excel export (styled): every list, Uang Makan, Workshop Ops, Rekap Absensi, KPI, financial reports (letterhead, bordered header, number format, total row, signature block "Kota, <long date> … <name>").
- Print: Quotation (Unit/PS), Sales Order, Item Requisition (no prices), PO, Goods Receipt, Delivery Order, Invoice, BAPP, Surat Tugas, Inspection Report, payslip, leave form, Rekap Absensi, financial statements. All user strings HTML-escaped.

---

# 11. KNOWN PITFALLS (lessons from the blueprint's bug-fix history — design these out)

1. Whole-collection overwrite races → lost stock, vanished POs/IRs, "approved but not applied". **Row-level atomic RPCs only.**
2. Comparing by object identity after a background refresh (`indexOf(editingQ)`) silently dropped edits. **Always match by primary key.**
3. Quotation-revision families: SO → quotation lookups must resolve through the family (3 separate bugs).
4. Stock reserved by raw sum double-counted; use FIFO allocation. Reservation must derive from **active SOs' full need**, not just IRs (SOs covered by stock never created an IR and got double-claimed).
5. DO never decremented stock and could ship with 0 stock. Decrement atomically; hard-block shortfall.
6. Dinas Luar double-counted inside Riil; half-day Ijin leaked into Absen and wiped Uang Kerajinan; manual negative-Ijin hacks. Use explicit override fields as specified in 7.8.
7. Case-sensitive name matching hid 90% of a sales manager's clients; use IDs/usernames, not display names.
8. Stale cached HTML after deploy → users ran old code. No-cache on the shell; show build id.
9. Per-part reservation recomputation froze Master Part (30 s). Bulk/DB computation.
10. Profile updates blocked by RLS looked like success (empty result). Always check affected-row count and surface errors.
11. Default password = username+"123", client-side SHA-256, `Math.random` IDs, unescaped print HTML (external audit findings). Do not repeat.
12. Hard-coded person names in logic (Epy, Pipin, Jhon, Septi, Asil, Stefanus, Laurent, HO1/HO2/HO3). **Never hardcode people; use roles and configurable username lists.**
13. Hard-coded bank account in invoice print. Make it a setting.
14. Optimistic UI with swallowed errors diverged from the server. Always show failures.
15. Revenue not recognized when approval step was removed. When a policy change skips a step, move any side-effect (auto-post) to the surviving step.

---

# 12. BUILD PLAN (phases — wait for my "continue" after each)

**Phase 0 — Foundations:** repo, Vite/TS, Supabase project + migrations framework, theme/component library, DialogHost, auth + profiles + RLS skeleton, roles/modules model, sidebar/groups, Dashboard shell, Company Settings, Manajemen User, audit log + activity log (triggers), document numbering RPC, edit-request + two-gate delete engine, notification badge engine.
**Phase 1 — Masters & Sales:** Master Part (+BOM, subcategories, import), Master Unit, Master Mill, Customer (groups, rating, access), Supplier, Quotation Unit + PS (revisions, termin, approvals, print), Sales Activity, Sales Order (routing), Client History.
**Phase 2 — Procure-to-Stock-to-Ship:** Item Requisition + FIFO allocation, PO Local/Overseas, Goods Receipt + Internal DO, Parts Use + returns, Delivery Order, Mutasi Barang, Document Trail, Price Analysis (sales & purchase), Client Analysis.
**Phase 3 — Field/Workshop:** Mekanik Report suite (visit, DAR, part rec, status, BAPP, surat tugas, inspection templates), Maintenance Schedule, Uang Dinas, Ticketing, Workshop Ops, Workshop Dashboard, KPI Mekanik.
**Phase 4 — Finance:** Invoice (3 modes, payments, rating feed), Jurnal, Bank Recon, Financial Dashboard, LK (COA, saldo awal, GL, reports, mapping, auto-post, fixed assets), Master COA.
**Phase 5 — HR & KPI:** Attendance + Rekap Mesin import/correction, Form Cuti, Uang Makan, Payroll (+ overtime, BPJS, payslips), User Analysis, Komisi, KPI Sales.
**Phase 6 — Hardening:** RLS test suite per role, load test with 5k parts / 20k documents, mobile pass, a11y pass, backup/restore runbook, deploy docs, optional legacy data migration + numbering migration tool.

Each phase must end with: migrations applied, seeds, tests green, a short demo script, and an updated `docs/MODULES.md`.

---

# 13. ACCEPTANCE CHECKS (must pass before declaring done)

1. **Golden path:** sales_manager drafts Unit Quotation → submits → GM approves → sales_admin creates SO (BOTH) → inventory creates IR for shortfall → purchasing creates PO → GM approves → storage receives at HO (status `Diterima HO - Menunggu Konfirmasi Workshop`, Internal DO created, **stock unchanged**) → logistic confirms at workshop (**stock +qty**) → logistic creates DO (**stock −qty**, blocked when insufficient) → finance creates Invoice from termin (DP then Pelunasan) → records payments → Jurnal/Bank Recon/LK entries exist and balance → Document Trail shows the whole chain.
2. **Concurrency:** two browsers approve different edit-requests / receive goods on the same part simultaneously → no lost update; both audit entries present.
3. **FIFO:** 3 SOs compete for 2 units of stock → total shortfall equals demand − stock; only queue losers get IR.
4. **Revision family:** revise an approved quotation that already has an SO → the revision is not offered for another SO (or follows partial-SO rules) and Follow-Up lists don't resurrect it.
5. **Permissions (automated per role):** every role can open only its modules; Payroll/User Analysis hard locks hold even with custom_modules; mekanik sees only own Uang Dinas; sales_manager sees only assigned clients — verified at the **database** level.
6. **Delete:** non-owner delete creates a pending Hapus request; GM approval applies it; owner deletes directly; linked rows cleaned.
7. **Client-facing prints** (Quotation, SO, Invoice, DO) contain no internal description and no sales-discount kickback.
8. **Totals/commission/overtime/absence/payroll** unit tests match worked examples in the spec (write them from Section 8 and 7.8).
9. **Journal engine:** unbalanced lines → no post; missing mapping → no post and no exception; re-fired event replaces, never duplicates.
10. **Mobile (375 px):** every list renders as cards; every form usable; no horizontal scroll.
11. **Security checks:** no default passwords; HTML-escaped prints; RLS on every table; no secrets in repo.
12. **Docs:** README with deploy steps, `.env.example`, `docs/*` complete.

---

# 14. FIRST ACTION

Do **not** write code yet. Start now with protocol step 1: read the Brief, and if anything marked `{{...}}`/`ASK` is unfilled, send me ONE batched question list (with defaults). Then step 2 (domain mapping table) and step 3 (plan), and wait for my approval.

=== END PROMPT ===
