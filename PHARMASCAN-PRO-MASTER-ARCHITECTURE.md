# 🏥 PharmaScan Pro — Master Architecture & Requirement Document
**Boots Oasis (Store 30762) | Dubai, UAE**
**Version:** 6.0 · Architect Edition
**Date:** March 2026

---

## 1️⃣ PROJECT UNDERSTANDING SUMMARY

### 1.1 What This App Is
PharmaScan Pro is a **mobile-first Progressive Web App (PWA)** used by pharmacy staff at Boots Oasis (Store 30762, Dubai) to track medicine and product expiry dates. Staff scan product barcodes (GS1 / EAN-13 / QR) using the phone camera, record expiry dates and batch numbers, then export the data for compliance review.

### 1.2 Who Uses It
- Pharmacy floor staff (daily scanning operations)
- Store manager / supervisor (reviewing exports, bulk approvals)
- Location: Dubai, UAE — products carry Arabic/English labels, GCC barcodes

### 1.3 Current State (Extracted from Project Files)
The project has **two overlapping versions** with inconsistencies:

| Aspect | Version A (index.html / PharmaScan Pro v5.0) | Version B (DETAILED-PROMPT.md / PharmaTrack) |
|--------|----------------------------------------------|----------------------------------------------|
| App Name | PharmaScan Pro | PharmaTrack |
| Scanner Library | QuaggaJS | BarcodeDetector API |
| Nav Tabs | Scanner + History | Scan + History + Paste |
| Settings | Bottom sheet | Hamburger slide-out menu |
| Security | Not implemented | PIN: 9633 |
| Export | Not implemented in HTML | TSV/CSV custom column order |

### 1.4 Real Data Snapshot (Boots_Oasis_Expiry_Tracker.xlsx)
- **Total Records:** 94 items
- **Coverage Period:** Jan–Jun 2026 (6 months)
- **Status Distribution:**
  - SOLD: 15 items (15.9%)
  - REMOVED: 6 items (6.4%)
  - PENDING (no remarks): 73 items (77.7%)
- **Expiry Distribution:** Jan-26 (25), Feb-26 (13), Mar-26 (7), Apr-26 (18), May-26 (18), Jun-26 (13)

---

## 2️⃣ COMPLETE FEATURE LIST

### 2.1 Barcode Scanning
| Feature | Priority | Source |
|---------|----------|--------|
| Camera-based real-time scanning | CRITICAL | index.html |
| Manual barcode entry (keyboard) | CRITICAL | index.html |
| GS1 DataMatrix parsing | HIGH | DETAILED-PROMPT.md |
| GS1-128 parsing | HIGH | DETAILED-PROMPT.md |
| EAN-13, EAN-8 | HIGH | All files |
| QR Code | MEDIUM | DETAILED-PROMPT.md |
| UPC-A, UPC-E | MEDIUM | DETAILED-PROMPT.md |
| Image upload scanning | MEDIUM | DETAILED-PROMPT.md |
| Paste bulk GS1 strings (one per line) | HIGH | DETAILED-PROMPT.md |

### 2.2 GS1 Application Identifier (AI) Parsing
| AI Code | Field | Format |
|---------|-------|--------|
| 01 | GTIN | 14 digits |
| 17 | Expiry Date | YYMMDD → auto-fill expiry field |
| 10 | Batch/Lot Number | auto-fill batch field |
| 21 | Serial Number | stored but not exported |
| 30 | Quantity | auto-fill quantity field |

### 2.3 Product Lookup Chain
1. **Exact match** → master database by GTIN-14
2. **Last-8 match** → match on last 8 digits of GTIN
3. **Seq-6 match** → match on 6-digit sequence
4. **External API lookup** (if online):
   - Brocade.io (medicines): `https://www.brocade.io/api/items/{GTIN14}`
   - Open Food Facts: `https://world.openfoodfacts.org/api/v2/product/{barcode}.json`
   - UPCitemdb: `https://api.upcitemdb.com/prod/trial/lookup?upc={barcode}`
5. **Unknown Product** → manual entry form

### 2.4 Inventory Data Fields
| Field | Source | Format | Required |
|-------|--------|--------|----------|
| RMS | Manual entry | alphanumeric | Optional |
| DESCRIPTION | Master DB / API / Manual | text | Required |
| BARCODE (GTIN) | Scanned / Manual | 8–14 digits | Required |
| QNTY | User input | integer ≥ 1 | Required (default: 1) |
| EXPIRY | GS1 AI-17 / Manual | YYYY-MM (display) / DDMMYY (export) | Required |
| BATCH | GS1 AI-10 / Manual | alphanumeric | Optional |
| LOCATION NO | Store config | numeric | Auto: 30762 |
| LOCATION NAME | Store config | text | Auto: BOOTS OASIS |
| REMARKS | Manual / action | SOLD / REMOVED / null | Optional |
| SCAN_TIMESTAMP | System | ISO-8601 datetime | Auto |
| MATCH_TYPE | System | EXACT / LAST8 / SEQ6 / API / MANUAL | Auto |

### 2.5 Expiry Status Logic
| Status | Condition | Color | Visual |
|--------|-----------|-------|--------|
| EXPIRED | Days ≤ 0 | #ef4444 (Red) | Red text + strikethrough row |
| EXPIRING SOON | Days 1–90 | #f59e0b (Orange) | Orange text |
| GOOD | Days 91+ | #10b981 (Green) | Green text |
| NO DATE | No expiry | #6b7280 (Gray) | Gray text |

### 2.6 Quantity Auto-Merge
- When scanning: check if **same GTIN + same BATCH** already exists
- If match: **increment quantity** by 1, update timestamp, show "+1 added (total: X)"
- If new: create new entry

### 2.7 Status Actions
- Mark as **SOLD** → updates REMARKS field, changes row visual
- Mark as **REMOVED** → updates REMARKS field, changes row visual
- Mark as **PENDING** → clears REMARKS field (default state)

### 2.8 Export Format
```
Column Order: RMS | BARCODE (GTIN) | DESCRIPTION | EXPIRY (DDMMYY) | BATCH | QUANTITY | LOCATION NO | LOCATION NAME | REMARKS
```
- TSV format (tab-separated, Excel-ready)
- CSV format (comma-separated, quoted)
- Filename: `boots-oasis-{YYYYMMDD}.tsv`

### 2.9 Master Data Management
- Upload CSV/TSV with at minimum Barcode + Product Name columns
- Auto-detect delimiter (comma / tab / semicolon)
- Auto-detect column headers
- Two modes: REPLACE (clear + reload) or APPEND (add without clearing)
- PIN required for upload
- Master data persists in IndexedDB

### 2.10 Security
- PIN code: **9633** (4 digits)
- Required for: edit history, upload master, restore backup, clear all history
- PIN entry UI: 4-dot progress, numeric keypad, shake on wrong PIN

### 2.11 Haptic Feedback
| Event | Pattern |
|-------|---------|
| Navigation / button | 10ms |
| Scanner start | 30ms |
| Successful scan/save | [30,50,30]ms |
| Error / wrong PIN | [100,50,100]ms |

### 2.12 PWA Requirements
- Service Worker with cache-first strategy
- Offline-capable after first load
- Installable (Add to Home Screen)
- IndexedDB for all persistent data
- Manifest with icons (72–512px)

### 2.13 Backup & Restore
- Export full backup as JSON (history + master data)
- Restore from JSON backup (PIN required)
- Backup format: `{ version, app, exportDate, history[], master[] }`

### 2.14 Summary Dashboard
- Total items count
- Total quantity sum
- Items expiring (≤90 days)
- Items expired
- Breakdown by expiry month
- Status breakdown (SOLD / REMOVED / PENDING)

---

## 3️⃣ FULL DATA MODEL

### 3.1 IndexedDB: `pharmascan_db` (v6)

#### Store: `history`
```javascript
{
  id: "uuid-v4",                  // Primary key
  gtin14: "06297001031292",       // GTIN padded to 14 digits
  gtin13: "6297001031292",        // GTIN without leading zero
  rms: "850007144067",            // RMS code (optional)
  description: "EDYST TAB 20MG 4S",
  barcode: "6297001031292",       // Raw scanned barcode
  quantity: 2,
  expiry: "2026-01",              // YYYY-MM format stored
  expiryDDMMYY: "012026",         // DDMMYY for export
  batch: "BG24025A",
  locationNo: 30762,
  locationName: "BOOTS OASIS",
  remarks: "SOLD",                // SOLD | REMOVED | null
  matchType: "EXACT",             // EXACT | LAST8 | SEQ6 | API | MANUAL
  serialNo: null,                 // GS1 AI-21 if present
  scanTimestamp: "2026-01-15T10:30:00Z",
  editTimestamp: null,
  gs1Parsed: true,                // Was GS1 string parsed?
  gs1Raw: "(01)06297001031292(17)260131(10)BG24025A"
}
```
- **Index:** `gtin14` (for lookup)
- **Compound Index:** `[gtin14, batch]` (for auto-merge)
- **Index:** `expiry` (for filtering)
- **Index:** `remarks` (for status filtering)

#### Store: `master`
```javascript
{
  id: "uuid-v4",
  gtin14: "06297001031292",       // Primary key alternative
  gtin13: "6297001031292",
  name: "EDYST TAB 20MG 4S",
  brand: "",
  supplier: "",
  category: "",
  rms: "",
  uploadBatch: "2026-01-15",      // When this was uploaded
  source: "CSV_UPLOAD"            // CSV_UPLOAD | API | MANUAL
}
```
- **Index:** `gtin14`
- **Index:** `gtin13`
- **Index:** `last8` (last 8 digits of gtin14)

#### Store: `config`
```javascript
{
  key: "storeInfo",
  value: {
    storeNo: 30762,
    storeName: "BOOTS OASIS",
    pin: "9633",
    apiLookupEnabled: true,
    vibrationEnabled: true,
    theme: "dark"
  }
}
```

#### Store: `apiCache`
```javascript
{
  gtin14: "06297001031292",       // Key
  data: { name, brand, matchType },
  cachedAt: "2026-01-15T10:30:00Z",
  ttl: 86400000                   // 24h in ms
}
```

### 3.2 Master Data CSV → DB Mapping
```
CSV Column Aliases → Internal Field
barcode / gtin / ean / upc / code → gtin14
name / product / description / item → description
brand / manufacturer → brand
supplier / vendor → supplier
rms / rms_code → rms
```

---

## 4️⃣ UI STRUCTURE

### 4.1 Navigation Architecture
```
App
├── Header (fixed)
│   ├── Logo + Version
│   ├── Store Badge (30762 · BOOTS OASIS)
│   └── Actions: [☰ Menu] [📤 Export]
│
├── Bottom Nav (3 tabs)
│   ├── 📷 SCAN
│   ├── 📋 HISTORY
│   └── 📊 DASHBOARD
│
└── Floating Action Button (Camera Scan) — above bottom nav
```

### 4.2 Screens

#### SCAN Screen
- Input field (barcode entry)
- [📷 Camera] [✏️ Manual] buttons
- Stats strip: Items | Total Qty | Expiring
- Recent Scans list (last 5)

#### HISTORY Screen
- Search bar (filter by any field)
- Filter chips: [All] [Expired] [Expiring] [Good] [SOLD] [REMOVED]
- Sort: [Newest] [Expiry Date] [Product Name]
- List of history cards (color-coded by expiry status)
- Each card: Product Name, GTIN, Expiry, Batch, Qty, Status badge, Edit/Actions

#### DASHBOARD Screen
- Summary stats grid (4 cards: Total Items, Total Qty, Expiring, Expired)
- Expiry breakdown by month (bar chart or list)
- Status breakdown (SOLD / REMOVED / PENDING)
- Quick export button

#### Camera View (Full Screen Overlay)
- Video feed
- Scanning frame with animated line
- Close button
- Status text: "Position barcode within frame"

#### Bottom Sheets (Modal)
1. **Scan Sheet** — product details + expiry/batch/qty form
2. **Unknown Product Sheet** — manual name entry form
3. **Settings Sheet** — store info, DB stats, file upload
4. **Bulk Paste Sheet** — textarea for multi-line paste
5. **PIN Entry Sheet** — 4-dot display + numeric keypad

#### Side Menu (Hamburger Slide-out)
- Master Data Stats
- Upload CSV (Replace / Append)
- Export TSV
- Export CSV
- Download Backup JSON
- Restore Backup
- Clear All History (danger)
- App Version

### 4.3 History Card Layout
```
┌─ [STATUS COLOR BAR] ───────────────────────────────────┐
│ EDYST TAB 20MG 4S                    ×2  [SOLD]        │
│ 6297001031292  ·  Exp: Jan 2026  ·  BG24025A           │
│ RMS: 850007144067           📝 Edit  ⋯ Actions          │
└────────────────────────────────────────────────────────┘
```
- Left border color = expiry status color
- Expired rows: strikethrough on product name
- SOLD rows: lighter opacity
- REMOVED rows: cross-out + red tint

---

## 5️⃣ WORKFLOW DIAGRAMS

### 5.1 Scan Workflow
```
User scans/enters barcode
        ↓
Parse GS1 string?
  YES → Extract GTIN (AI-01), Expiry (AI-17), Batch (AI-10), Qty (AI-30)
  NO  → Raw barcode → pad to GTIN-14
        ↓
Search master DB (EXACT → LAST8 → SEQ6)
  FOUND → Open Scan Sheet (pre-filled)
  NOT FOUND → Check online?
    YES → Call APIs (Brocade → OpenFoodFacts → UPCitemdb)
      FOUND → Open Scan Sheet (partially filled)
      NOT FOUND → Open Unknown Product Sheet
    NO (offline) → Open Unknown Product Sheet
        ↓
User confirms/edits fields
        ↓
Check GTIN + BATCH already in history?
  YES → Increment quantity → Update timestamp → Toast "+1 added"
  NO  → Create new entry → Toast "Item saved"
        ↓
Save to IndexedDB → Update UI stats
```

### 5.2 Export Workflow
```
User clicks Export
        ↓
Fetch all history from IndexedDB
        ↓
Map to export schema:
  RMS | BARCODE | DESCRIPTION | EXPIRY(DDMMYY) | BATCH | QNTY | LOCATION NO | LOCATION NAME | REMARKS
        ↓
Convert expiry: "2026-01" → "012026" (DDMMYY with 01 for day)
        ↓
Generate TSV string with tab delimiters
        ↓
Trigger browser download: boots-oasis-{YYYYMMDD}.tsv
```

### 5.3 Status Update Workflow
```
User taps history card → [⋯ Actions]
        ↓
Action menu: Mark as SOLD | Mark as REMOVED | Clear Status | Delete
        ↓
PIN Modal → Enter 9633
  CORRECT → Apply status → Update IndexedDB → Refresh list
  WRONG   → Shake animation → Error haptic → Retry
```

### 5.4 Master Data Upload Workflow
```
User opens ☰ Menu → Upload CSV
        ↓
PIN Modal → Enter 9633
        ↓
File picker → Select CSV/TSV file
        ↓
Parse file → Auto-detect delimiter
        ↓
Map columns (barcode, name, brand, supplier)
        ↓
Preview: "Found 450 products. Mode: [Replace] [Append]"
        ↓
Confirm → Save to master IndexedDB store
        ↓
Rebuild lookup indexes
        ↓
Toast: "✓ 450 products loaded"
```

---

## 6️⃣ NEW PWA ARCHITECTURE

### 6.1 System Architecture
```
┌─────────────────────────────────────────────────────────┐
│                     PharmaScan Pro v6                    │
├─────────────────────────────────────────────────────────┤
│  UI Layer (HTML5 + CSS3 + Vanilla JS ES2022)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │  Scan    │ │ History  │ │Dashboard │ │Settings  │  │
│  │  Module  │ │  Module  │ │  Module  │ │  Module  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────────────────┤
│  Core Services Layer                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │  Scanner │ │   GS1    │ │ Product  │ │  Export  │  │
│  │ Service  │ │  Parser  │ │ Matcher  │ │ Service  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────────────────┤
│  Data Layer                                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │  IndexedDB (pharmascan_db v6)                    │  │
│  │  Stores: history | master | config | apiCache    │  │
│  └──────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│  Network Layer (Online only, graceful degradation)       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│  │Brocade   │ │OpenFood  │ │UPCitemdb │               │
│  │  .io API │ │Facts API │ │  API     │               │
│  └──────────┘ └──────────┘ └──────────┘               │
├─────────────────────────────────────────────────────────┤
│  PWA Layer                                               │
│  ┌────────────────────┐ ┌──────────────────────────┐   │
│  │  Service Worker    │ │  Web App Manifest         │   │
│  │  Cache Strategy:   │ │  display: standalone      │   │
│  │  Cache-First       │ │  orientation: portrait    │   │
│  └────────────────────┘ └──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 6.2 File Structure (v6)
```
boots-oasis-pharmascan/
├── index.html              ← Single page, all UI
├── app.js                  ← App entry point + module imports
├── modules/
│   ├── db.js               ← IndexedDB wrapper
│   ├── scanner.js          ← Camera + GS1 parsing
│   ├── matcher.js          ← Product lookup chain
│   ├── history.js          ← History CRUD + merge logic
│   ├── export.js           ← TSV/CSV/JSON export
│   ├── upload.js           ← CSV parsing + master upload
│   ├── api.js              ← External API integrations
│   ├── pin.js              ← PIN security module
│   ├── ui.js               ← UI helpers, toasts, sheets
│   └── dashboard.js        ← Summary stats
├── styles/
│   ├── main.css            ← Base styles + CSS variables
│   ├── components.css      ← Cards, sheets, buttons
│   └── animations.css      ← Transitions, scan line
├── sw.js                   ← Service Worker
├── manifest.json           ← PWA manifest
└── icons/
    ├── icon-72.png → icon-512.png
```

### 6.3 Technology Decisions
| Component | Technology | Reason |
|-----------|------------|--------|
| Framework | Vanilla JS (ES2022) | No build step, offline-safe, fast |
| Scanner | BarcodeDetector API (primary) + QuaggaJS (fallback) | Native API is faster; QuaggaJS for Safari |
| Storage | IndexedDB (via idb wrapper) | Structured, indexed, large capacity |
| State | Module-level reactive state (no Redux) | Simple, no dependency |
| CSS | CSS Custom Properties + Grid/Flexbox | Theme-able, no framework needed |
| PWA | Service Worker (Workbox patterns) | Reliable caching |
| Export | Blob + URL.createObjectURL | Works offline |

### 6.4 Service Worker Cache Strategy
```
Cache Name: pharmascan-v6-cache

CACHE_FIRST (always serve from cache):
  - index.html
  - app.js + all modules/*.js
  - styles/*.css
  - icons/*
  - Google Fonts

NETWORK_FIRST (try network, fallback cache):
  - External API calls (Brocade, OpenFoodFacts, UPCitemdb)

NO_CACHE (always network):
  - Nothing critical — all app assets are cached
```

### 6.5 Security Architecture
```
PIN Module:
  - PIN stored in IndexedDB config store (not localStorage)
  - Default: "9633"
  - Hashed with SHA-256 before storage
  - Rate limiting: 3 wrong attempts → 60s lockout
  - PIN required for: edit | upload master | restore | clear history | change PIN

Protected Routes:
  - History Edit → PIN
  - Status Change (SOLD/REMOVED) → PIN
  - Master Upload → PIN
  - Backup Restore → PIN
  - Clear All → PIN + confirmation text ("DELETE ALL")
```

### 6.6 Performance Optimizations
- **Lazy render history** — render 20 items, load more on scroll (IntersectionObserver)
- **Debounce search** — 300ms debounce on history search input
- **IndexedDB cursors** — use cursor iteration for large datasets, not .getAll()
- **API caching** — cache API results 24h in apiCache store
- **Image scanning** — offload to Web Worker if available
- **Animation** — use `will-change: transform` on animated elements
- **Fonts** — preconnect + display=swap

---

## 7️⃣ IMPLEMENTATION PLAN

### Phase 1 — Core MVP (Week 1)
- [ ] index.html base structure + CSS variables
- [ ] IndexedDB setup (db.js) with all stores
- [ ] Scanner module (camera + GS1 parser)
- [ ] Manual entry flow
- [ ] History display with color-coded expiry
- [ ] Export to TSV (Boots format)
- [ ] PWA manifest + Service Worker

### Phase 2 — Smart Features (Week 2)
- [ ] Master data upload (CSV/TSV)
- [ ] Product matcher (EXACT → LAST8 → SEQ6)
- [ ] External API lookup chain
- [ ] Quantity auto-merge logic
- [ ] PIN security system
- [ ] Status actions (SOLD / REMOVED)

### Phase 3 — Enhanced UX (Week 3)
- [ ] Dashboard with stats + charts
- [ ] Bulk paste processing
- [ ] Backup/restore JSON
- [ ] Haptic feedback
- [ ] Settings panel
- [ ] Search + filter + sort history
- [ ] Edit history entries

### Phase 4 — Polish (Week 4)
- [ ] Animations + transitions
- [ ] Error handling (offline, API fails)
- [ ] Performance optimization
- [ ] Cross-device testing (iOS Safari, Android Chrome)
- [ ] Install prompt UI

---

## 8️⃣ FUTURE SCALABILITY PLAN

### Short Term (3–6 months)
- **Multi-store support** — same app, switch between store profiles
- **Cloud sync** — optional Supabase/Firebase sync across devices
- **PDF report export** — generate formatted compliance PDF
- **Barcode image upload** — scan barcodes from product photos

### Medium Term (6–12 months)
- **Team accounts** — staff login, audit trail (who scanned what)
- **Notifications** — push alerts when items approach expiry (7 days, 1 day)
- **Supplier mapping** — auto-link products to supplier database
- **Analytics dashboard** — monthly expiry trends, high-risk products

### Long Term (12–24 months)
- **API integration with Boots HQ** — sync data upstream
- **Computer vision** — OCR expiry date directly from product packaging
- **Reorder alerts** — link expiry tracking to procurement
- **Multi-language** — Arabic UI for local staff
- **Regulatory reporting** — UAE MOHAP compliance report generation

---

## ⚠️ CLARIFICATION REQUIRED

1. **PIN customization** — Should staff be able to change the PIN from 9633, or is it fixed per store?
2. **Location field** — Is LOCATION NAME always "BOOTS OASIS" or can items be in different sections/aisles?
3. **GTIN barcode inconsistency** — In the Excel, some barcodes start with letters (e.g., "A939251157") — are these internal RMS codes masquerading as barcodes?
4. **Duplicate rows in Excel** — Several rows have identical GTIN + BATCH (e.g., HYALO4 SKIN CREAM appears 3 times with same batch G04620). Is this intentional (different scan events) or data error?
5. **Export format — EXPIRY column** — The spec says DDMMYY but expiry is stored as YYYY-MM (month-only). Should day be always "01"? E.g., 2026-01 → "012601"?
6. **API rate limits** — UPCitemdb has 100 requests/day free tier. Should we implement a daily counter and warn users?
7. **Offline API caching TTL** — 24h seems reasonable but should it be configurable?
8. **Image icons** — The uploaded icons show a pill/capsule design (dark background, teal pill). Should the app icon stay this exact design?

---

*Document generated by Senior PWA Architect — all requirements extracted from: index.html, DETAILED-PROMPT.md, API-INTEGRATION-GUIDE.md, FREE-BARCODE-DATABASES-GUIDE.md, manifest files, and Boots_Oasis_Expiry_Tracker.xlsx (94 records)*
