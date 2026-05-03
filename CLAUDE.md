# Pre-Pro Paperwork — Claude Context

This folder contains standalone browser-based tools built for entertainment lighting pre-production, specifically for the **BMW 2026** show (and reusable for future shows). All tools are single-file HTML apps with no build step, no server, and no dependencies — open directly in Chrome. State is saved to `localStorage`. Files are tracked in the GitHub repo: `https://github.com/KillingTheMains/Pre-Pro-Paperwork.git`

The working folder on disk is:
`~/jason@killingthemains.com - Google Drive/My Drive/Cowork Playground/Pre-Pro Paperwork/`

**Live site:** https://pre-pro.netlify.app/
Netlify auto-deploys on every push to `main`. No manual deploy step needed.

**After every code change, commit and push via desktop-commander** (the user should never have to run git manually):
```bash
cd ~/"jason@killingthemains.com - Google Drive/My Drive/Cowork Playground/Pre-Pro Paperwork"
# clear lock if needed: rm .git/index.lock
git add rack-builder/index.html   # (or whichever file changed)
git commit -m "description of change"
git push
```
Note: `rack-builder/index.html` and `loom-builder/index.html` are the repo copies (always kept in sync with `Rack Builder.html` / `Loom Builder.html` in the same folder).

---

## Tool 1 — Rack Builder

**File:** `Rack Builder.html` (also `rack-builder/index.html` in repo)
**~2700 lines** — vanilla JS, dark entertainment-industry theme, monospace font (SF Mono / Fira Code).

### What it does
Visual rack layout planner. You build named color-coded racks, populate them with equipment, assign DMX universes to ports, map snake/loom cables to ports, and link ports to each other across racks. Includes a print-to-PDF layout export.

### App state model
Everything lives in one object `S`:
```js
S = {
  project: {
    name, version,
    racks: [ Rack ],
    snakes: [ Snake ],
    customTypes: [ EquipmentType ]
  },
  modal: null | { type, data },
  activeRackId: string
}
```
Persisted to `localStorage` key `'rack_builder_project'`. `save()` writes `S.project`. `render()` rebuilds the full DOM from `S`.

### Rack object
```js
{
  id, name, color,   // color is a hex from COLORS array
  equipment: [ Item ]
}
```

### Item (equipment) object
```js
{
  id, typeId, label, ip, notes,
  portLabels: {
    [portId]: {
      label,       // destination label, e.g. "USL FLR"
      notes,
      universe,    // DMX universe string, e.g. "UNIV 5"
      connRackId,  // port-to-port connection
      connItemId,
      connPortId,
      snakeId,     // snake break-in connection
      snakeLineId
    }
  }
}
```

### Built-in equipment types (BUILTIN_TYPES)
Each type has: `id, name, ruHeight, color, hasIp, ports[]`
Each port has: `id, name, type` (type = `'input'` | `'output'` | `'network'` | `'power'`)

| typeId | Name | RU | Key ports |
|---|---|---|---|
| `luminode12` | LumiNode 12 | 1 | 1× network input, 12× DMX outputs |
| `luminode4` | LumiNode 4 | 1 | 1× network input, 4× DMX outputs |
| `lumisplit210` | LumiSplit 2.10 | 1 | 2× DMX inputs, 10× DMX outputs |
| `optosplitter` | Optosplitter (1×5) | 1 | 1× DMX input (`in_1`), 5× DMX outputs (`out_1`–`out_5`) |
| `proplex1616` | ProPlex IQ 1616 | 2 | 1× network, 16× DMX inputs, 16× DMX outputs |
| `gigacore10` | GigaCore 10 | 1 | 10× network ports |
| `gigacore16` | GigaCore 16 | 2 | 16× network ports |
| `cpc4_breakout` | CPC4 Breakout | 1 | 1× CPC4 input, 4× DMX outputs |
| `cpc8_breakout` | CPC8 Breakout | 1 | 1× CPC8 input, 8× DMX outputs |

### Snake (loom cable) object
```js
{
  id, name,
  location,   // physical location, e.g. "TRUSS B1" — auto-fills position on lines
  type,       // 'cpc4' | 'cpc8' (connector type)
  color,      // hex
  lines: [ SnakeLine ]
}
```

### SnakeLine object
```js
{
  id, lineNum,
  position,     // physical position label; defaults to snake.location when empty
  universe,     // DMX universe string — if contains "SP" → spare line (pink row, no firstFixture)
  firstFixture, // first fixture number on this universe
  connRackId,   // break-in connection to a rack port
  connItemId,
  connPortId
}
```

### Color system
```js
const COLORS = ['#4a9eff','#3dd68c','#a855f7','#f97316','#ec4899','#22d3ee','#facc15','#ef4444','#f0f0f0','#9ca3af','#14b8a6'];
const COLOR_NAMES = {
  '#4a9eff':'BLUE', '#3dd68c':'GREEN', '#a855f7':'PURPLE',
  '#f97316':'ORANGE', '#ec4899':'PINK', '#22d3ee':'CYAN',
  '#facc15':'YELLOW', '#ef4444':'RED', '#f0f0f0':'WHITE',
  '#9ca3af':'GREY', '#14b8a6':'TEAL'
};
```
Pink (`#ec4899`) is reserved for spare cables throughout the system.

### BMW 2026 snake inventory (pre-loaded)
13 original Breakout snakes (named A–M, no `location` field) + 42 loom snakes imported from LOOM LIST spreadsheet (all have a `location` field — this is used to distinguish them in migration logic).

Loom snakes: B1/B1 SP, B2/B2 SP, B3/B3 SP, B4/B4 SP (TRUSS B1–B4, CPC4), MA/MA SP/MB/MB SP (TRUSS M, CPC8), W/W SP (TRUSS W, CPC8), O/O SP (ORCHESTRA, CPC4), C1–C9 + SPs (CAR SALON, CPC4), Q1–Q4 + SPs (Q1–Q4 FLOOR, CPC4).

### Key functions
- `uid()` — generates unique IDs
- `h(s)` — HTML-escapes a string
- `save()` — writes `S.project` to localStorage, no re-render
- `render()` — full DOM rebuild from state (avoid calling inside blur/input handlers — use targeted DOM updates instead)
- `connInfo(rackId, pl)` — returns `{ short, full, crossRack, type }` for a port's connection badge
- `snakeLineConnInfo(line)` — same for snake lines
- `portLabel(port)` — returns `"PORT " + port.name`
- `eqLabel(e)` — returns `"TYPENAME · Label"` (e.g., `"NODE · Node 1"`)
- `saveInlineField(el)` — blur handler for inline spreadsheet inputs; saves to data model + calls `_applySpareRowDOM()` if field is `universe`
- `_applySpareRowDOM(univEl, sid, lid, line)` — targeted in-place DOM swap for spare/non-spare row (avoids full re-render killing focus)
- `slKeyNav(event, el)` — Tab/Enter navigation across inline snake fields
- `copyUniversesToSpare(rackId, itemId)` — copies universe assignments from main equipment to matching spare

### Critical pattern: inline editing without re-render
Snake line fields (position, universe, firstFixture) use `onblur="saveInlineField(this)"`. These must NOT call `render()` — it destroys the DOM and breaks keyboard navigation. Only `save()` + targeted DOM manipulation.

### Port row HTML structure (two-line layout)
```html
<div class="port-row">
  <div class="prow-top">
    <span class="pdot [type][port-connected?]"></span>
    <span class="pname">PORT 1</span>
    <span class="pedit">✏</span>
  </div>
  <!-- conditional: only if lab || ci || univBadge -->
  <div class="prow-bot">
    <span class="puniv">UNIV 5</span>       <!-- DMX ports only -->
    <span class="plabel">USL FLR</span>     <!-- destination label -->
    <span class="pconn pconn-local/remote/snake">→ GREEN RACK · Node 1 · PORT 3</span>
  </div>
</div>
```

### Connection display format
- In rack view badge: `RACK NAME · Equipment Label · PORT X`
- In modal current-connection label: `→ Rack Name › Equipment Label › PORT X`
- In dropdowns: equipment = `"TYPENAME · Label"`, ports = `"PORT X"` or `"PORT X — UNIV 5"`

### Migration pattern
`init()` runs `migrateLoomSnakes()` — compares existing project snakes against `defaultProject().snakes` filtered by `.location` presence. Missing loom snakes are appended without overwriting existing data.

### Rack visual design
Network rack aesthetic: SVG cage-nut hole pattern on rails via CSS `background-image` data URI, cap bars with screw icons, RU number badges on left rail, right rail column on each row. Max rack width 1400px.

---

## Tool 2 — Loom Builder

**File:** `Loom Builder.html` (also `loom-builder/index.html` in repo)
**~1450 lines** — vanilla JS, dark theme matching Rack Builder palette.

### What it does
Loom list planner and cable tracker. Tracks all cables for a show — their type, physical parts (segment lengths), locations, loom groups, breakout connectors, and colors. Multiple shows can be stored. Exports to CSV and Lightwright cable string format.

### App state model
```js
shows = {
  [showId]: ShowObject
}
activeId = string   // currently selected show
```
Persisted to `localStorage` key `'loom_builder_shows'`. Active show ID in `'loom_builder_active'`.

### ShowObject
```js
{
  show: { name, client, venue, date, version, notes },
  locations: [ Location ],
  cables: [ Cable ]
}
```

### Location object
```js
{ id, name, color, secColor }
```

### Cable object
```js
{
  id,
  locationId,   // links to a Location
  label,        // cable name/identifier
  type,         // cable type string (see below)
  breakout,     // breakout connector type (optional)
  loomId,       // loom group name (free text, e.g. "A LOOP", "DD DROP")
  color,        // primary color name (from COLOR_NAMES)
  secColor,     // secondary color name
  parts,        // array of 6 segment length strings (feet)
  notes
}
```

### Cable types
`SOCAPEX`, `CPC8`, `CPC4`, `2X DMX`, `DMX 2X`, `QM-8`, `FIBER`, `ETHERCON`, `CAT5`, `CAT6`, `POWER`, `OTHER`

Parts hints by type:
- SOCAPEX: Pt1=from distro, Pt2=trunk run, Pt3–6=on-truss/drops
- CPC8/CPC4: Pt1=from distro, Pt2=trunk

### Color badges (type colors)
```js
SOCAPEX: {bg:'#0a84ff', fg:'#fff'}
CPC8:    {bg:'#ff6b00', fg:'#fff'}
CPC4:    {bg:'#ffcc00', fg:'#000'}
// others use neutral styling
```

### Tabs
Overview · Cables · Build View · Inventory · Settings

- **Overview**: stats (total cables, by type counts, location breakdown)
- **Cables**: searchable/sortable table of all cables with inline edit
- **Build View**: grouped by location with parts breakdown
- **Inventory**: breakout connector counts, cable length summaries
- **Settings**: show metadata (name, client, venue, date, version, notes), location management

### Key functions
- `newShowObj(name)` — creates a blank show object with empty arrays
- `getShow()` — returns the active show
- `markDirty()` / `persistSave()` — debounced auto-save to localStorage
- `renderAll()` — redraws all tabs
- `saveCable()` — reads the cable form and saves/creates a cable
- `exportCSV()` — downloads cable list as CSV
- `exportInventoryCSV()` — downloads inventory summary as CSV
- `exportLightwright()` — downloads Lightwright-compatible cable strings
- `exportShow()` / `importShow()` — JSON backup/restore of a full show

---

## Pre-Pro Suite architecture

### localStorage keys
| Key | Owner | Contents |
|---|---|---|
| `pps_suite` | Landing page | `{ showName, client, venue, year }` — suite-wide show identity |
| `loom_builder_shows` | Loom Builder | All shows data; active show is source of truth for cable list |
| `loom_builder_active` | Loom Builder | ID of currently active show |
| `rb_project` | Rack Builder | Rack layout, snakes (with `loomCableId`), equipment, connections |

### Data ownership
- **Show name / identity** (`pps_suite`) — set on landing page, synced to/from Loom Builder's active show name
- **Cable list** — owned by Loom Builder (Overview page is the master cable manager). Rack Builder reads from it via `syncFromLoom()`.
- **Rack layout / universe assignments** — owned by Rack Builder only
- **Loom segment lengths / cable groups** — owned by Loom Builder only

### Cable sync (Rack Builder ← Loom Builder)
`syncFromLoom()` runs on Rack Builder `init()` and on every `storage` event for `loom_builder_shows`. It:
1. Reads the active Loom Builder show's cables
2. Matches each cable to a snake by `loomCableId` (or by `name` for migration)
3. Updates snake identity fields: `name`, `type`, `color`, `location`
4. Resizes the `lines[]` array if type changed (CPC4↔CPC8), preserving existing universe data
5. Creates new snakes for cables that don't yet have one (empty line data)
6. Marks snakes whose Loom cable was deleted as `orphaned: true` (dimmed in sidebar, data preserved)

### Color mapping (Loom → Rack Builder hex)
```js
LOOM_COLOR_TO_HEX = { RED:'#ef4444', ORANGE:'#f97316', YELLOW:'#facc15', GREEN:'#3dd68c',
  BLUE:'#4a9eff', PURPLE:'#a855f7', GREY:'#9ca3af', TEAL:'#14b8a6',
  WHITE:'#f0f0f0', PINK:'#ec4899', CYAN:'#22d3ee', ... }
```

### Cross-tab live sync
`window.addEventListener('storage', ...)` in both apps:
- Rack Builder re-runs `syncFromLoom()` when `loom_builder_shows` changes
- Both apps update their show name display when `pps_suite` changes

### Snake `loomCableId` field
Each Rack Builder snake that originated from Loom Builder has `loomCableId` pointing to the Loom Builder cable's `id`. Snakes without this field are rack-only (legacy or manually added).

---

## Shared conventions

- **Dark theme** throughout: deep navy/charcoal backgrounds, accent colors per entity type
- **Monospace font**: SF Mono, Fira Code, ui-monospace
- **Spare = pink**: `#ec4899` is always the spare/backup color across both tools
- **SP in universe name** → marks a spare line in Rack Builder (pink row, no firstFixture)
- **No frameworks**: vanilla JS only, no npm, no build step
- **Single-file HTML**: CSS, JS, and HTML all in one file per tool
- **localStorage persistence**: no server, no accounts
- **Color hex values are canonical**: always use the COLORS array values, not ad-hoc hex codes

## Viewport height pattern (REQUIRED for all new tools)

CSS `height: 100%` on `body` requires the full chain `html → body → #root → …` to all be properly sized, which breaks silently. **Always use one of these two approaches instead:**

### Option A — `100vh` on body (preferred for simpler layouts)
```css
body { height: 100vh; display: flex; flex-direction: column; overflow: hidden; }
.main-scroll-area { flex: 1; overflow-y: auto; }
```

### Option B — JS override (required when a `#root` div is the render target)
When `render()` rebuilds `#root` innerHTML, `flex: 1` on children of `#root` doesn't work unless `#root` is itself a flex container. Use both CSS and JS:

```css
html, body { height: 100%; overflow: hidden; margin: 0; }
body { display: flex; flex-direction: column; }
#root { flex: 1; min-height: 0; display: flex; flex-direction: column; overflow: hidden; }
#app-body { display: flex; overflow: hidden; /* height set by JS */ }
#sidebar { overflow-y: auto; flex-shrink: 0; }
#main-scroll { flex: 1; overflow-y: auto; }
```

```js
function applyViewportHeight() {
  const appBody = document.getElementById('app-body');
  if (!appBody) return;
  const header = document.getElementById('app-header');
  const used = header ? header.getBoundingClientRect().height : 0;
  appBody.style.height = (window.innerHeight - used) + 'px';
  appBody.style.maxHeight = (window.innerHeight - used) + 'px';
}
window.addEventListener('resize', applyViewportHeight);
// Call applyViewportHeight() at the end of every render() call
```

**Rack Builder uses Option B. Loom Builder uses Option A.**

## Reference docs (in `reference-docs/`)
- GigaCore 10 user manual
- ProPlex IQ 1616 user manual  
- LumiNode 12 spec sheet
- LumiSplit 2.10 spec sheet
- XSP/XSR spec sheet

## Roadmap
- Shared project data / cross-tool linking (snake connections visible in Loom Builder)
- Network layout planner
- Patch sheet / universe map generator
- Full pre-pro workflow dashboard
