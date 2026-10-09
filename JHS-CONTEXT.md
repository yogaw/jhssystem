# JHS System — Master Context Document

> Compiled: 2026-10-09, from live-site analysis (https://jhssystem.vercel.app) — JS bundle reverse-engineering + Supabase data inspection.

---

## 1. What Is JHS System?

A **web-based business management system for PT Saeti Beton Pracetak (SBP)** — an Indonesian **precast/pretensioned concrete pile manufacturer** ("beton pracetak"). The system manages the full commercial pipeline: sales leads → price quotations → approval → signed PDF → archive.

- **App**: JHS System **v0.1** (single-page React app, SPA)
- **Client company**: PT Saeti Beton Pracetak (SBP) — product is **Square Piles (SQP)** with **pile joints** (the "JHS joint" — the system is named after it)
- **Developed by**: Pinion Pte Ltd. & PT Pinion Teknologi Indonesia (licensed to SBP)
- **Domain**: foundation piles for construction projects (malls, power plants, ports…)
- **Two factories**: **Cikarang** and **Surabaya** (`factory_origin`)

### User's stated scope (the "module penawaran")
1. Pembuatan dokumen penawaran (quotation document creation) ✅ exists
2. Approval workflow (CCO → CEO) ✅ exists
3. Digital signature (not highly secure) ✅ exists — PNG signature upload, stamped into PDF
4. PDF viewer ✅ exists — client-side PDF review (jsPDF-generated)
5. Archive ✅ exists — `SLS_lead_documents` + `documents` storage bucket

### Required databases (user's list → actual tables)
| User's list | Actual implementation |
|---|---|
| Database material (bahan baku) | `materials`, `concrete_job_mix`, `sqp_material_rebars`, `labor_rates` |
| Database product (barang jadi) | `sqp_product_needed`, `sqp_product_price_history`, `sqp_joint_needed`, `sqp_joint_price_history`, `sqp_bill_of_quantities` |
| Database vendor/klien | `PRC_vendors`, `PRC_vendor_contacts` (new, empty) + `SLS_clients` |
| Database user | `profiles` (+ Supabase `auth.users`) |
| Database projects | `SLS_projects` |
| Database surat (document mgmt) | `SLS_lead_documents` + Supabase Storage bucket `documents` |

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React (create-react-app), React Router v6, plain CSS-in-JS (inline styles), Inter + Georgia fonts, Tabler Icons webfont |
| PDF | **jsPDF 2.5.1** (client-side generation, loaded from cdnjs) |
| Backend | **Supabase** (Postgres + Auth + Storage + PostgREST + RPC) |
| Supabase project | `ztynayfcycucetuclqvi` (`https://ztynayfcycucetuclqvi.supabase.co`) |
| Hosting | Vercel (`jhssystem.vercel.app`, static build, last deployed 2026-10-02) |
| External data | `api.frankfurter.dev` (USD→IDR FX rates), `americasoilwatch.com` (Brent oil price, Stooq + EIA fallback) — used for material price trend widgets |
| Auth | Supabase email/password (`/auth/v1/token?grant_type=password`), role-based route guards |

### Access / credentials
- **URL**: https://jhssystem.vercel.app/dashboard (login at `/login`)
- **Admin login**: `johannes@bepinion.com` / `Pinion2026!`
- Generic reset password for users (admin RPC): `JHS2026!`

---

## 3. Roles & Navigation

Role labels (Indonesian UI):
`admin`=Administrator, `sales`=Sales, `marketing`=Marketing, `cco`=**Direktur Komersial**, `ceo`=**Direktur Utama**, `procurement`=Pengadaan, `lab`=Laboratorium, `qc`=QC

Sidebar config per role (exact, from bundle):

```
admin:       Dashboard, Penjualan[Daftar Prospek, Daftar Penawaran, Harga Produk], Marketing, Pengadaan[Log Survey Harga, Vendor], Laboratorium, QC
sales:       Dashboard, Penjualan[Daftar Prospek(/sales), Harga Produk]
marketing:   Dashboard, Penjualan[Daftar Penawaran(/marketing/quotations), Harga Produk]
cco:         Dashboard, Penjualan[Daftar Penawaran(/cco/quotations), Harga Produk]
ceo:         Dashboard, Penjualan[Daftar Penawaran(/ceo/quotations), Harga Produk]
procurement: Dashboard(/procurement "Dashboard Pengadaan"), Pengadaan[Log Survey Harga, Vendor]
lab:         Dashboard, Laboratorium   → PAGE IS A PLACEHOLDER ("Modul Laboratorium — segera hadir.")
qc:          Dashboard, QC             → PAGE IS A PLACEHOLDER ("Modul QC — segera hadir.")
```
All roles also get mobile bottom-nav + **Profil** (`/settings`) for profile, avatar & signature upload.

---

## 4. Routes

| Route | Purpose |
|---|---|
| `/login`, `/reset-password` | Auth |
| `/dashboard` | Per-role dashboard (targets w/ carry-over, margin summaries, sales totals) |
| `/sales` | Daftar Prospek (lead list) |
| `/sales/create`, `/sales/:leadID`, `/sales/:leadID/edit` | Lead create/detail/edit |
| `/leads/:leadID/pricing` | Pricing engine per lead (SQP costing wizard) |
| `/marketing` | Marketing home |
| `/marketing/leads/:leadID/pricing` | Marketing variant of pricing |
| `/marketing/quotations` | Penawaran Marketing (quotation list/creation) |
| `/cco/quotations`, `/cco/quotations/:quotationID` | CCO quotation review/approval |
| `/ceo/quotations`, `/ceo/quotations/:quotationID` | CEO final approval |
| `/procurement` | Dashboard Pengadaan |
| `/procurement/log-survey-harga` | Log Survey Harga (vendor price surveys) |
| `/procurement/vendor` | Vendor management (CRUD + contacts) |
| `/products` | Harga Produk (product price catalog + history) |
| `/lab`, `/qc` | ⚠️ placeholders — "segera hadir" (coming soon) |
| `/settings` | Profil (avatar, signature, phone) |

---

## 5. Database Schema (Supabase Postgres, public schema)

Row counts as of 2026-10-09. PKs are UUIDs.

### Sales (SLS_)
- **`SLS_leads`** (11) — leadid, stakeholderid, projectid, leadstatus, createdby, pic (uuid→profiles), delivery (bool), delivery_type (`franco`…), deliverycost, factory_origin (`cikarang`/`surabaya`), shipment_cost_per_ton, outside_java_delivery_label/total, attentionto, quotationlanguage (`id`/`en`), validityperiod ("30 days"), submissionnote, total_sales, createdat
- **`SLS_projects`** (5) — projectid, projectname, projecttype (null so far), project_loc_city, project_loc_full_address, createdat
- **`SLS_clients`** (18) — clientid, clientname, clientaddress, createdat. *Clients are big Indonesian SOE/private contractors: Waskita, WIKA, Adhi Karya, Hutama Karya, PP, Jaya Konstruksi, Pertamina EP, PLN Indonesia Power, Pelindo, Summarecon…*
- **`SLS_lead_stakeholders`** (25) — stakeholderid, clientid, projectid, stakeholderrole, createdat
- **`SLS_lead_documents`** (6) — documentid, leadid, doctype (`Purchase Order`/`Kontrak`/`Lainnya`), filename, filesize, mimetype, storagepath (→ bucket `documents`), uploadedby, createdat — **this is the archive**
- **`SLS_location_city`** (514) — cityid, cityname, provincename, is_java (bool), lat, lng (used for delivery-distance calc vs factory)
- **`SLS_monthly_targets`** (1) — id, month, year, target_amount, achieved_amount, carry_over (dashboard: target penjualan; carry-over rolls to next month)

### Marketing / Quotations (MKT_)
- **`MKT_price_quotations`** (28) — quotationid, leadid, quotationnumber (format **`0002/PH/SBP/MKT/00`**), quotationversion, versiondate, status, grandtotalharga, totalprodukbetonprice, totaljointprice, produkbetonmargin, jointmargin, totalmargin, notes (long Indonesian T&C boilerplate), notes_template_version, marketing_note, sales_note, draft_pdf_path, final_pdf_path, submissionnote, submissionnote_role, createdat
- **`MKT_price_qoutations_approval`** (22) ⚠️ *table name has typo "qoutations"* — approvalid, quotationid, reviewerrole (**`CCO`** then **`CEO`**), reviewername, revieweruserid, result (`approved`/`rejected`/null=pending), resultreason, reviewedat, createdat. Upsert on `(quotationid, reviewerrole)`; pending CEO record is deleted when CCO rejects.
- **`MKT_land_delivery_rate`** (14) — id, factory_origin, min_km, max_km, rate_per_ton (Rp), is_port_rate, notes (e.g. 0–50 km Cikarang = 50.000/ton "Jabodetabek"; 101–200 km = 120.000 "Cirebon, Semarang area")

### Procurement (PRC_) — *module baru, tables still empty*
- **`PRC_vendors`** (0) — vendor_id, vendor_name, location, …
- **`PRC_vendor_contacts`** (0)
- **`PRC_survey_harga`** (0) — material_id, vendor_id, unitprice, created_at (FKs: `fk_logsurvey_vendor`, `fk_logsurvey_material`)

### Product / costing engine (sqp_*) — SQP = Square Pile
- **`sqp_product_needed`** (31) — productneededid, leadid, product_id, product_type (`SQP`), mutubeton (K-500), grade (A/B/C), dimension, length, segment (`SGL`/`TOP`/`MID`/`BOT`), compression_or_tension (`CO`/`TN`), joint_type, productneededqty, price, customrequest, custom_base_cost, createdat
- **`sqp_joint_needed`** (31) — sqpjointneededid, productneededid (FK), qty, proposedprice, customrequest
- **`sqp_bill_of_quantities`** (1.312) — **44 columns**: full costing BOM per pile config — pile_width_mm, pile_length_m, pile_depth_mm, pile_grade, pile_segment, mutu_beton, pile_concrete_strength_mpa, volume_m3_per_meter, joint_design (`JHS Catalogue`/`JHS Custom`/`Endplate Catalogue`/`Endplate Custom`), jhs_b/d/e/f/g_mm, number_of_strands, strand_weight_kg, shear_reinforcement_*, confinement_*, anchors (edge/middle: rod_id, count, length_m, weight_kg), shoes_hairpins, end_plate_*, wedge_weight_kg, tie_wire (kawat bendrat) weight, plate_joint_weight_kg, material_reinforcement_id…
- **`sqp_product_price_history`** (656) — per-config unit prices: price, price_per_m, price_per_pile, source (`auto_refresh`, …), recorded_at/by, factory_origin (`Cikarang`)
- **`sqp_joint_price_history`** (492) — joint prices (price, price_per_set) by config
- **`sqp_material_rebars`** (23) — Material_Reinforcement_ID, type, diameter_mm, quality name, yield/ultimate strength MPa, weight kg/m
- **`additional_material_needed`** (0, empty)

### Master data
- **`materials`** (28) — material_id, material_type (Semen/PC Strand/Pasir/Split…), material_grade, material_diameter, unit (kg/m3), is_active
- **`concrete_job_mix`** (26) — per factory_origin + grade (K500): material_category (semen/pasir/split/air/admixture), material_specs, quantity_per_m3, selected_concrete_material_id, pile_width_min/max_mm
- **`labor_rates`** (6) — per pile_width×height (250…500 mm), basis_year: production_rate_per_m1, jhs_joint_catalogue_per_set, jhs_joint_custom_per_set, endplate_joint_catalogue_per_set, endplate_joint_custom_per_set, additional_strand_per_strand_per_m1, pembesian_per_kg (Rp 800/kg)
- **`profiles`** (19) — id (=auth.users.id), full_name, role, phonenumber, signature_path, avatar_path, created_at. Storage buckets: `profile-signatures` (PNG), `profile-avatars`

### RPCs
- `check_email_exists` — signup validation
- `reset_user_password(target_user_id, new_password)` — admin resets any user to `JHS2026!`

### Product ID naming convention
`SQP{width_cm}_{length_m}_{grade}_{CO|TN}_{SGL|TOP|MID|BOT}_{EP|JHS}_{K500}`
e.g. `SQP250_1_A_CO_BOT_EP_K500` = square pile 25×25 cm (written 250), 1 m length grade, Grade A, compression, bottom segment, endplate joint, concrete K500. ~328 distinct configs in price history.

---

## 6. Business Workflow (as implemented)

```
Sales creates LEAD (client, project, stakeholder/PIC, factory origin,
delivery type & city, validity, quotation language id/en)
   │  status: draft → sent_to_marketing
   ▼
MARKETING builds pricing at /leads/:id/pricing
   - picks SQP products (width, length, grade, CO/TN, segment, joint type)
   - BOQ auto-computed (concrete mix from concrete_job_mix, rebars, strands,
     joints, plates, anchors, wire) + labor_rates
   - margins computed: joint margin, produk beton margin, total margin
   - delivery cost from MKT_land_delivery_rate (km from factory, Rp/ton)
   ▼
PRICE QUOTATION created (numbered NNNN/PH/SBP/MKT/NN)
   - draft PDF generated client-side (jsPDF) → draft_pdf_path (bucket documents)
   - status: draft → requesting_approval
   ▼
APPROVAL WORKFLOW (MKT_price_qoutations_approval, sequential)
   1) CCO (Direktur Komersial) reviews at /cco/quotations
      approve → CEO record created; reject → record result + reason, CEO row deleted
   2) CEO (Direktur Utama) final-approves at /ceo/quotations
   ▼
APPROVED → final PDF (with digital signature PNG stamped via jsPDF addImage)
   → status approved/sent → archived in SLS_lead_documents
   ▼
Lead outcome: won / lost / closed (terminated for quotations)
```

### Status enums (from data + code)
- **Lead**: `draft`, `sent_to_marketing`, `requesting_approval`, `approved`, `sent`, `won`, `lost`, `closed`
- **Quotation**: `draft`, `requesting_approval`, `approved`, `rejected`, `sent`, `terminated`
- **Approval result**: `approved`, `rejected`, `null` (pending)
- **Lead doc types**: `Purchase Order`, `Kontrak`, `Lainnya`
- **Joint designs**: `JHS Catalogue`, `JHS Custom`, `Endplate Catalogue`, `Endplate Custom`

### Dashboard widgets observed
- "Ringkasan Margin Penawaran" (quotation margin summary grid)
- "Total Penjualan Pile", "Total Penjualan Joint", "Nilai Proyek", "Total Keseluruhan"
- "Tren Harga Material" (Brent oil + USD/IDR FX charts)
- Monthly target vs achieved + **carry_over** (from `SLS_monthly_targets`)
- Procurement: "Dashboard Pengadaan", "Vendor bulan ini", "Vendor aktif", "Vendor Baru"

---

## 7. Data Snapshot (2026-10-09)

19 users, 18 clients, 5 projects (*Pembangunan Mall di Banten, TRCH, Wisma Nusantara, Mall Makassar, Mall AEON Tanjung Barat*), 11 leads, 28 quotations, 22 approvals, 6 archived documents, 28 materials, 514 cities, ~1.3k BOQ rows, 656 product prices, 492 joint prices. Example target: Aug 2026 — target Rp 15 M, achieved Rp 90.17 M (carry-over 75.17 M).

Users (roles): Johannes (admin), Adi Wijaya (ceo), Javon Bog & Siswo Wicaksono (cco), Hans Canisto & Stefanus Rudy (marketing), Hans/Yosafat/Fahmi (sales), Lucia (procurement), Melisa (lab), Ivana (qc).

---

## 8. Quirks & Gotchas

1. ⚠️ Approval table is misspelled: **`MKT_price_qoutations_approval`** ("qoutations") — keep the typo when integrating.
2. `factory_origin` values are lowercase in leads (`cikarang`, `surabaya`) but capitalized in price history (`Cikarang`) — inconsistent casing across tables.
3. Lab & QC modules are **placeholders** ("segera hadir") — roles exist, pages empty.
4. Vendor module (PRC_*) is **brand new** — tables created & UI shipped, zero rows.
5. Digital signatures are simple PNG uploads (`profile-signatures` bucket) stamped into the PDF — intentionally not cryptographically secure (per user).
6. PDF generation is fully client-side (jsPDF); only the file path is stored in Postgres, binaries live in Storage bucket `documents`.
7. UI language is Indonesian; quotation documents can be `id` or `en` (`quotationlanguage`).
8. Some quotations/leads are test data (e.g. quotation #0002 "terminated", Mall AEON project), app is at v0.1.
9. Money is IDR (Rp), formatted with `id-ID` locale; labor_rates `basis_year: 2031` suggests escalator assumptions built into pricing.

---


