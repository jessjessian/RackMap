# 🗺️ RackMap — System Sketchpad
**Lee Huat Frozen Goods Warehouse · 利发冷冻货仓**
_Last updated: 2025 · Single-page reference for the whole system_

---

## 1. Quick Facts

| Item | Value |
|---|---|
| **Live URL** | `irene-ong.github.io/rackmap` |
| **Repo** | `github.com/irene-ong/rackmap` → file `index.html` on `main` |
| **Backend** | Supabase project **"Rack MAP"** (`xndxogfqaddzzyegtmzl.supabase.co`) |
| **Auth URL** | `supabase.com/dashboard/project/xndxogfqaddzzyegtmzl/auth/users` |
| **Stack** | Single HTML file + Supabase (Postgres + Auth) + SheetJS for Excel export |
| **Size** | ~177 KB, ~124 JS functions, all in one `index.html` |

---

## 2. Roles & Logins

| Role | Person | Email | Password | Core Duty |
|---|---|---|---|---|
| ⚙️ **Developer** | Irene (KLCII) | `ongirene@klc.edu.sg` | _contact Irene_ | System maintenance, SKU mgmt, tech support |
| 📋 **Sales Admin** | Sales team | `sales@leehuatseafood.com` | `LeeHuatSales@26` | Enters received items into Zone Z; receiving exceptions; **issues delivery orders** |
| 🏭 **Warehouse** | Ops team | `Leehuat.ops@gmail.com` | `LeeHuatOps#26` | Count, place (physical + system), **Ji**, weight edits, splits, **delivery deductions** |
| 👑 **Boss** | Jess | `yunzhen@leehuatseafood.com` | `LeeHuat` | Full access: view stock, download reports |

> ⚠️ Emails are case-insensitive (stored lowercase). Passwords **are** case-sensitive.
> All roles see the same live data — differences are procedural, not technical permissions.

**Adding a user:** Supabase dashboard → Authentication → Add user → ✅ Auto Confirm. If login fails, run:
```sql
UPDATE auth.users SET email_confirmed_at = NOW(), confirmed_at = NOW()
WHERE email = 'new@email.com';
```

---

## 3. Warehouse Layout (Zones)

| Zone | Meaning | Columns | Shelves |
|---|---|---|---|
| **R1–R20** | Main right racks | A–F | Top / Middle / Bottom |
| **L1–L20** | Left racks | A–B | Top / Middle / Bottom |
| **#01-22** | Specific numbered zone | — | matches physical numbering |
| **Zone Z (New Incoming)** | ⭐ Staging area for all new stock | single slot | — |

**Flow:** Sales Admin enters new stock into **Zone Z** → Warehouse physically places on shelf **and** drags in RackMap from Zone Z to the matching rack → Zone Z empties.

Rendering: racks are **3 per row**, fixed **320px** width, `flex-wrap`. All layout is baked as **inline styles** so cached CSS can't collapse the grid.

---

## 4. Core Concepts

### Batch
The atomic unit of stock. One SKU on one shelf can have multiple batches. Each batch has:
`batchId · arrival date · qty · weightList[] · jiList[]`

### Available vs Total
> **Header number on each shelf = qty − Ji.** Consigned (寄) units are **excluded** from available everywhere: shelf header, left-panel Avail, Move popup, stats bar.

### 寄 Ji (Consignment)
Goods **sold but still stored** awaiting customer pickup. Fully managed by **Warehouse**.
- Up to **3 companies** per batch, each with qty + company name + invoice no.
- Shown as orange badge `寄 X (N家)`; **hover** shows company — qty per line.
- Excluded from available stock. Cannot delete/reduce a batch below its Ji qty.

### Weight variations
Each batch supports up to **3 weight variations**, each with its own quantity.
- Entering weights+quantities **splits into one batch per weight** (separate rows, shared arrival date).
- The weight quantities **must sum exactly** to the batch total, or it won't save.
- Leave all qty at 0 → single batch using the Variation-1 weight text.

---

## 5. Key Actions (What / Where / Who)

| Action | Where | Who |
|---|---|---|
| **Enter new stock** | Zone Z slot → Assign popup → SKU + date + qty (+ weights) | 📋 Sales Admin |
| **Place on shelf** | Drag from Zone Z → target rack (matches physical) | 🏭 Warehouse |
| **Move stock** | ⇄ button → up to **3 destinations** in one go (FIFO, Ji stays behind) | 🏭 Warehouse |
| **Add batch** | Shelf card "+ add batch" or Item Detail → Add Batch | 🏭 Warehouse |
| **Edit qty** | Shelf card qty box or −/+ ; Item Detail qty box | 🏭 Warehouse |
| **Edit weight** | ⚖️ on shelf card **or** Item Detail Weight cell | 🏭 Warehouse |
| **✂️ Split batch** | Item Detail → ✂️ → up to 3 weight/qty pairs | 🏭 Warehouse |
| **Mark 寄 Ji** | Orange 寄 button → up to 3 customers | 🏭 Warehouse |
| **Deduct delivery** | − button per sales delivery order | 🏭 Warehouse |
| **Fix receiving mismatch** | Batch qty box (Zone Z or rack) | 📋 Sales Admin only |
| **Manage SKUs** | Top toolbar → SKU Manager | ⚙️ Dev / 📋 Sales Admin |
| **Search** | Top search bar (code or name) | anyone |

---

## 6. Reports (Excel)

| Button | Location | Sheets | Use |
|---|---|---|---|
| 📊 **导出 Export** | Top toolbar (green) | 4: Stock, Batch, History, Grid | Quick daily operational export (English) |
| 📊 **管理报告 Mgmt Report** | Stats bar | 5: Stock Summary, Batch Detail, **Ji Report**, Change History, Grid Map | Full bilingual management report, Ji separated |
| 🟠 **寄 X 件 ↓ Ji Report** | Stats bar (when Ji exists) | 1: Ji items, **one row per customer** | Consignment billing / audit |

Ji Report and Mgmt Report break out each Ji customer on its own row (company + invoice + qty).

---

## 7. Key Rules

1. ⏰ **Sales Admin entry:** within 24 hrs before delivery — not earlier.
2. 📅 **Warehouse placement:** within 2 working days — physical **and** system.
3. 🚫 **Receiving corrections:** Warehouse cannot self-edit → WhatsApp Sales Admin + photos.
4. 📦 **Ji:** Warehouse-managed, up to 3 companies, excluded from available.
5. ⚖️ **Multi-weight:** weight quantities must equal the batch total exactly.
6. 🔄 **Physical shelf position must match RackMap position exactly.**

---

## 8. Database Schema (Supabase)

**Table `assignments`** (one row per batch)

| Column | Type | Notes |
|---|---|---|
| `shelf_id` | text | which shelf |
| `sku_code` | text | product code |
| `batch_id` | text | unique per batch |
| `arrival` | date | arrival date |
| `qty` | int | total qty in batch |
| `weight` | text | joined label e.g. `10kg / 12kg` |
| `weight_list` | **jsonb** | `["10kg","12kg","15kg"]` |
| `ji_qty` | int | total Ji (legacy/summary) |
| `ji_company` | text | first company (legacy) |
| `ji_invoice` | text | first invoice (legacy) |
| `ji_list` | **jsonb** | `[{q,c,i}, ...]` up to 3 |

**Required migration** (run once if not done):
```sql
ALTER TABLE assignments ADD COLUMN IF NOT EXISTS ji_list JSONB DEFAULT '[]'::jsonb;
ALTER TABLE assignments ADD COLUMN IF NOT EXISTS weight_list JSONB DEFAULT '[]'::jsonb;
```
> The app has a **graceful fallback** (`HAS_MULTI` flag): if these columns are missing, it writes only legacy single-value columns so nothing breaks — but multi-company/multi-weight only persist once the columns exist.

**Other tables:** `warehouse_structure` (zones/racks/shelves), `sku_catalog` (product list), `change_history` (audit log, last 200 loaded).

---

## 9. Update / Deploy Procedure

1. Open `github.com/irene-ong/rackmap` → `index.html` → pencil ✏️
2. `Ctrl+A` → delete → paste the entire new file
3. **Commit changes**
4. Wait 1–2 min (GitHub Pages rebuild)
5. On the live site: **Ctrl+Shift+R** (hard refresh)

**Recovery:** the last uploaded good version is always fetchable from
`raw.githubusercontent.com/irene-ong/rackmap/main/index.html`

---

## 10. Function Map (developer reference)

| Area | Key functions |
|---|---|
| **Load** | `loadStructure`, `loadSKUs`, `loadAssignments`, `loadHistory` |
| **Render** | `render`, `renderWarehouse`, `renderStats`, `renderSKUList`, `renderUnplacedList` |
| **Assign / Batch** | `openAssign`, `doAssign`, `openAddBatch`, `doAddBatch`, `collectWeightBatches` |
| **Qty** | `setBatchQty`, `changeBatchQty`, `setBatchQtyFromDetail`, `dbUpdateQty` |
| **Weight** | `openWeightModal`, `saveWtEdit`, `dbUpdateWeights` |
| **Split** | `openSplitModal`, `saveSplit` |
| **Ji** | `openJiModal`, `jiRecalc`, `saveJi`, `clearJi`, `dbUpdateJi`, `showJiTip` |
| **Move** | `openMoveBlock`, `updateMoveRack`, `updateMoveShelf`, `confirmMove` |
| **Zones** | `addRegularZone`, `addZoneZ`, `addRackToZone`, `confirmDeleteZone` |
| **Export** | `exportToExcel`, `exportFullReport`, `exportJiItems` |
| **DB core** | `dbAddBatch`, `dbDeleteBatch`, `dbUpdateJi`, `dbUpdateWeights`, `dbAddHistory` |
| **Helpers** | `parseJ`, `jiEntries`, `wtEntries`, `jiTipLabel`, `jiCoLabel`, `isMissingCol` |

---

## 11. Known Gotchas

- **Same SKU on multiple shelves** is normal — created by partial ⇄ Move or drag.
- **Ji-locked units never move** — Move/split always leave consigned qty behind; clear Ji first to move those.
- **Header shows available, not total** — if a shelf shows `0` with an orange `寄 60`, the stock is all consigned.
- **`node --check` clean** and **no orphan handlers** must hold after any code edit.
- When patching the HTML, define functions **before** the markup that references them (integrity check catches this).

---

_Contact: Irene Ong · ongirene@klc.edu.sg · KLC Reimagineers / Jedi Consulting_
