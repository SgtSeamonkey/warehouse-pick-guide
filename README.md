# warehouse-pick-guide

A self-contained, single-file browser pick guide that highlights yellow flow-rack sections and maps them to numbered orange cart slots. Open `index.html` in any modern browser — no server or internet connection required.

---

## How It Works

| Panel | Description |
|-------|-------------|
| **Left** | Orange 8-slot cart — shows cart slot numbers, part numbers, and descriptions for each slot |
| **Right** | Yellow flow rack (sections 1–38) — sections glow amber when active; each lit section shows the part number, description, and an orange badge with the cart slot number |

---

## Updating Pick Data — Find & Replace

All pick data lives in **one clearly marked block** at the top of the `<script>` section in `index.html`. Every value uses a `PLACEHOLDER_` string that you replace with real data using your text editor's **Find & Replace** (Ctrl+H / ⌘H).

### Step 1 — Orange cart slot parts (8 slots)

Find each placeholder and replace with the real part number / description:

| Find | Replace with |
|------|-------------|
| `PARTNUMBER_SLOT_01` | Real part number for cart slot 1 (max 30 chars) |
| `PARTDESC_SLOT_01` | Human-readable label for cart slot 1 (max 30 chars) |
| `PARTNUMBER_SLOT_02` … `08` | Part numbers for slots 2–8 |
| `PARTDESC_SLOT_02` … `08` | Labels for slots 2–8 |

### Step 2 — Yellow rack section parts (38 sections)

| Find | Replace with |
|------|-------------|
| `PARTNUMBER_RACK_01` | Real part number stored in rack section 1 (max 30 chars) |
| `PARTDESC_RACK_01` | Human-readable label for rack section 1 (max 30 chars) |
| `PARTNUMBER_RACK_02` … `38` | Part numbers for rack sections 2–38 |
| `PARTDESC_RACK_02` … `38` | Labels for sections 2–38 |

### Step 3 — Pick routes (which sections light up per cart slot)

Find the `PICK_ROUTES` object and replace the placeholder arrays with real rack section numbers. Each slot should have 5–7 section numbers.

```js
// Before (placeholder):
const PICK_ROUTES={
  1:[1,2,3,4,5],
  2:[6,7,8,9,10],
  ...
};

// After (real data example — cart slot 1 uses rack sections 14, 17, 21, 22, 28):
const PICK_ROUTES={
  1:[14,17,21,22,28],
  2:[1,3,5,7,9],
  ...
};
```

---

## Generating from SQL (SQL Server / T-SQL)

Run the query below against your database to produce the three sets of values to paste into `index.html`.

### Orange cart slot assignments

```sql
-- Returns part number and description for each of the 8 cart slots
SELECT
    sl.CartSlot,
    pt.PartNumber       AS PartNum,      -- replace PARTNUMBER_SLOT_xx
    pt.PartDescription  AS PartDesc      -- replace PARTDESC_SLOT_xx
FROM ActivePickList pl
JOIN PickLine sl ON sl.PickListID = pl.PickListID
JOIN Part     pt ON pt.PartID     = sl.PartID
WHERE pl.IsActive = 1
ORDER BY sl.CartSlot;
```

### Yellow rack section assignments

```sql
-- Returns part number and description for each rack section (1-38)
SELECT
    rl.RackSection      AS SectionNum,
    pt.PartNumber       AS PartNum,      -- replace PARTNUMBER_RACK_xx
    pt.PartDescription  AS PartDesc      -- replace PARTDESC_RACK_xx
FROM RackLocation rl
JOIN Part pt ON pt.PartID = rl.PartID
WHERE rl.IsActive = 1
ORDER BY rl.RackSection;
```

### Pick routes (which rack sections light up per cart slot)

```sql
-- Returns comma-separated rack section numbers per cart slot
SELECT
    sl.CartSlot,
    STRING_AGG(CAST(rl.RackSection AS VARCHAR), ',')
        WITHIN GROUP (ORDER BY rl.RackSection) AS RackSections
FROM ActivePickList pl
JOIN PickLine     sl ON sl.PickListID = pl.PickListID
JOIN RackLocation rl ON rl.PartID    = sl.PartID
WHERE pl.IsActive = 1
GROUP BY sl.CartSlot
ORDER BY sl.CartSlot;
```

Then copy the `RackSections` value for each slot into the matching array in `PICK_ROUTES`.

### Automated HTML generation (PowerShell example)

```powershell
# Read the template
$html = Get-Content .\index.html -Raw

# Replace orange cart parts (loop for all 8 slots)
$html = $html -replace 'PARTNUMBER_SLOT_01', '84KQ-A22600-AB'
$html = $html -replace 'PARTDESC_SLOT_01',   'Front Door Inner LH'
# ... repeat for slots 02-08

# Replace rack section parts (loop for all 38 sections)
$html = $html -replace 'PARTNUMBER_RACK_01', '84KQ-A01000-AB'
$html = $html -replace 'PARTDESC_RACK_01',   'Front Door Carrier LH'
# ... repeat for sections 02-38

# Save the final file
$html | Set-Content .\pick-guide-$(Get-Date -f yyyyMMdd).html
```

---

## Auto-Update (next 8 doors) Without Reloading

Your query script keeps rewriting `ORANGE_PARTS` and `PICK_ROUTES` in the HTML
file as before. **Stop reloading the page afterwards.** The open page re-reads
its own file every `CHECK_FOR_UPDATES_SECONDS` (default 10) and, only when the
doors or routes actually changed, redraws the orange cart and highlighted rack
sections in place. Rack edits, sizes, pictures and an open edit window are not
touched. This works whether the page is opened as a local file or from a server.

- Keep the file name and folder the same; the page re-reads the file it was opened from.
- The status dot turns red if the file could not be read.
- Set `CHECK_FOR_UPDATES_SECONDS=0` to turn it off.
- Other code on the page can also push data directly: `window.updatePickData({orange:{...},routes:{...}})`.

Rack edits, sizes and pictures are stored in the browser's IndexedDB, which has
room for many photos, so they also survive an actual reload. (The old
localStorage store held only ~5 MB and silently failed once pictures filled it.)

---

## Settings File (RH page)

In Chrome or Edge, click **📁 Link folder** in the header once and choose the
folder that holds `RH Door Line Kitting.html`. After that:

- **Every refresh** loads `warehouse-pick-RH-settings.json` from that folder
  (if it changed since this screen last loaded or saved it).
- **Export** saves `warehouse-pick-RH-settings.json` there and first copies the
  previous version to `warehouse-pick-RH-settings_backup_<date>_<time>.json`.
  To revert, rename a backup back to `warehouse-pick-RH-settings.json` and refresh.
- If this screen has edits that were never exported and the file changed
  elsewhere, the screen's version is saved as `..._backup_<date>_<time>_this-screen.json`
  before the file is loaded, so nothing is lost.
- If the browser asks again, the button turns yellow: **Reconnect folder** (one click).
- Without a linked folder (or in other browsers) Export downloads the file as before.

The LH page will use `warehouse-pick-LH-settings.json` the same way.

---

## Logo and Settings Monitor (RH and LH pages, Rev C)

- **Logo:** the Flex-N-Gate logo is built into both pages (white, top left), so
  no extra file is needed. To replace it, put a `flexngate-logo.png` (or `.svg`,
  white on transparent works best) next to the page. Source files are in
  `reference pics/`.
- **⚙ Settings** (header) opens a read-only live data monitor:
  - **Live status:** auto-update on/off, last check / successful read / data
    change, read errors, and data checks (placeholder door text, routes to
    sections that are not on the rack or have zero size, picked sections with
    no part text, screen edits that differ from the file).
  - **Orange parts:** `ORANGE_PARTS` and `PICK_ROUTES` per cart slot, rows that
    changed in the last minute tinted, plus the data as loaded.
  - **Rack parts:** every section with part, description, size, picture and the
    cart slots it feeds; where a saved screen edit differs from `RACK_PARTS` in
    the file, the file value is shown struck through.
  - **Event log:** every door-data change and read error since the page opened.
- The revision shows bottom left (`Rev C`). New code sits after the data, so
  `ORANGE_PARTS` is on line 264 and `RACK_PARTS` on line 279 in **both** the RH
  and LH pages (`PICK_ROUTES`: line 325 RH, 333 LH).
- The LH page has the same features as RH, including the folder settings file
  (`warehouse-pick-LH-settings.json`) and Rev B display fixes.

---

## Rack Section Reference

| Group | Sections | Physical location |
|-------|----------|-------------------|
| A | 1 – 8 | Far-left tall rack (4 rows × 2 cols) |
| B | 9 – 12 | Mid-left short rack, lower half only (2 rows × 2 cols) |
| C | 13 – 20 | Centre-left rack (4 rows × 2 cols) |
| D | 21 – 28 | Centre rack (4 rows × 2 cols) |
| E | 29 – 36 | Large right-centre area (big pigeon-hole bays) |
| F | 37 – 38 | Far-right narrow column (2 stacked) |

Each section square displays:
- **Section number** — small, top-left corner
- **Part number** — bold, auto-sized to fit the square (up to 30 characters)
- **Description** — italic, auto-sized below the part number (up to 30 characters)

---

## Distributing via Email

Attach `index.html` to your email. Recipients save it and open it in a browser (Chrome, Edge, Firefox). All styling, layout, and logic is self-contained — no internet connection is needed.

> **Note:** Email clients (Outlook, Gmail) block JavaScript. The pick guide only works when the HTML file is **opened directly in a browser tab**, not previewed inside the email client.
