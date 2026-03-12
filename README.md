# warehouse-pick-guide

A browser-based warehouse pick guide that highlights yellow flow-rack sections and maps them to numbered orange cart slots. Open `index.html` in any modern browser — no server required.

---

## How It Works

The right panel displays the yellow flow rack (sections 1–38). The left panel shows the 8-slot orange cart. When pick data is loaded, rack sections glow yellow and numbered orange badges appear in the corner of each lit section, telling the picker which cart slot the part belongs in.

---

## Demo Mode

Click **▶ Demo** in the header to cycle through four pre-built demo pick lists, each using all 8 cart slots with 5–7 rack sections illuminated per slot. Click **✕ Clear** to reset.

---

## Live Data via SQL / API

The guide polls `/api/cart-items` every 5 seconds when `CFG.useMock` is set to `false`. The endpoint must return a JSON array of exactly **8 objects** (one per cart slot, index 0 = slot 1). Any slot with no pick can be `null`.

### JSON payload format

```json
[
  { "description": "Front Seat Cushion", "rackSlots": [1, 2, 3] },
  { "description": "Rear Door Panel",    "rackSlots": [14, 15, 16, 17] },
  null,
  { "description": "Headliner",          "rackSlots": [29, 30, 31] },
  null,
  null,
  null,
  null
]
```

- **`description`** – part name shown in the footer ticker  
- **`rackSlots`** – array of rack section numbers (1–38) to illuminate for that cart slot; 5–7 sections recommended

### Enabling live data

In `index.html`, find the config object near the bottom of the `<script>` block and set `useMock` to `false`:

```js
const CFG = { apiEndpoint: '/api/cart-items', pollMs: 5000, useMock: false };
```

### Example SQL → JSON (SQL Server / T-SQL)

The query below reads an active pick list from your database and formats it as the required JSON array. Adjust table/column names to match your schema.

```sql
SELECT
    slot.CartSlot - 1  AS [index],   -- 0-based index matching cart slot 1-8
    part.PartDescription  AS [description],
    STRING_AGG(CAST(rack.RackSectionNum AS VARCHAR), ',') AS rackSlotsCsv
FROM PickList pl
JOIN PickListLine slot ON slot.PickListID  = pl.PickListID
JOIN Part         part ON part.PartID      = slot.PartID
JOIN RackLocation rack ON rack.PartID      = slot.PartID
WHERE pl.IsActive = 1
GROUP BY slot.CartSlot, part.PartDescription
ORDER BY slot.CartSlot;
```

Then in your middleware / API layer convert `rackSlotsCsv` to an integer array and emit the 8-element JSON array shown above. A minimal Node.js example:

```js
// GET /api/cart-items
app.get('/api/cart-items', async (req, res) => {
  const rows = await db.query(`/* SQL above */`);
  const result = Array(8).fill(null);
  rows.forEach(r => {
    result[r.index] = {
      description: r.description,
      rackSlots: r.rackSlotsCsv.split(',').map(Number)
    };
  });
  res.json(result);
});
```

### Rack section reference

| Group | Sections | Physical location |
|-------|----------|-------------------|
| A | 1 – 8 | Far-left tall rack (4 rows × 2 cols) |
| B | 9 – 12 | Mid-left short rack, lower half only (2 rows × 2 cols) |
| C | 13 – 20 | Centre-left rack (4 rows × 2 cols) |
| D | 21 – 28 | Centre rack (4 rows × 2 cols) |
| E | 29 – 36 | Large right-centre area (pigeon-hole shelves + conveyor) |
| F | 37 – 38 | Far-right narrow roller-conveyor (2 stacked) |

### Changing which sections illuminate

Edit the `rackSlots` arrays in the SQL output or directly in the `DEMOS` array inside `index.html` to point at any combination of section numbers 1–38.

---

## Distributing via Email

Save `index.html` and attach it to the email. Recipients open the file in their browser (Chrome, Edge, Firefox). All styling is self-contained — no internet connection is needed for the display to render correctly.

> **Note:** Email clients themselves (Outlook, Gmail web) block JavaScript, so the interactive features only work when the HTML file is opened directly in a browser tab.
