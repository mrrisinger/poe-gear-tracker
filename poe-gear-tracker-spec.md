# Path of Exile Gear Cost Tracker — Design Spec

## 1. Overview

A single-file HTML micro tool for tracking the cost of gear across Path of Exile leagues and characters. All data is stored in memory during use, with JSON save/load to the local filesystem. No server required.

---

## 2. Core Concepts

### 2.1 Data Hierarchy

```
Master Save File (.json)
└── League (e.g. "3.28 Mirage")
    └── Character (manually named)
        └── Item Slot
            ├── Active Item (0 or 1)
            └── Inactive Items (history)
```

### 2.2 Item Lifecycle

An item moves through these states:

1. **Active** — Currently equipped in a slot on a character.
2. **Inactive (Awaiting Sale)** — Removed from the slot but has no sell price. Appears in an "Awaiting Sale" list. Tracks which character used it last.
3. **Inactive (Sold)** — Has a sell price recorded. Moves to sold history.

**Transitions:**

- Adding a new item to an occupied slot → existing item becomes **Inactive (Awaiting Sale)**.
- Setting a sell price on any non-active item → item becomes **Inactive (Sold)**.
- Setting a sell price on an active item → item becomes **Inactive (Sold)**, slot becomes empty.
- Two-handed weapon placed in Weapon 1 → Weapon 2's active item (if any) becomes **Inactive (Awaiting Sale)**, Weapon 2 slot is locked/disabled.

### 2.3 Currency

- Each price is stored as a value + currency type: **Chaos**, **Divine**, or **Found** (zero cost).
- Prices are never auto-converted. They are stored in their original currency.
- The Reports page provides a Divine:Chaos ratio input to normalize all values to Divines for display.
- The Divine:Chaos ratio is persisted in the save file.

---

## 3. Item Slots

### 3.1 Main Gear Slots

Based on the PoE equipment screen. The visual layout approximates the in-game inventory using a 4-row × 6-column CSS grid with weapons spanning 2 rows vertically (matching their taller in-game appearance).

| Slot Name    | Grid Position |
|-------------|--------------|
| Helmet      | Row 1, center |
| Amulet      | Row 1, right of center |
| Weapon 1    | Rows 2–3, far left (spans 2 rows) |
| Body Armour | Rows 2–3, center (spans 2 rows) |
| Weapon 2    | Rows 2–3, far right (spans 2 rows) |
| Ring 1      | Row 2, left of center |
| Ring 2      | Row 2, right of center |
| Gloves      | Row 3, left of center |
| Belt        | Row 3, center |
| Boots       | Row 3, right of center |
| Flask 1–5   | Row 4, bottom row left to right |

**CSS grid definition:**
```css
grid-template-areas:
  ". helmet helmet amulet . ."
  "w1 r1 body body r2 w2"
  "w1 gloves belt belt boots w2"
  ". f1 f2 f3 f4 f5";
```

### 3.2 Skill Gem Slots

10 gem slots displayed as a vertical column on the **left** side of the gear grid, labeled Gem 1–10.

### 3.3 Jewel Slots

10 jewel slots displayed as a vertical column on the **right** side of the gear grid, labeled Jewel 1–10.

### 3.4 Two-Handed Weapon Handling

When an item is added to Weapon 1 and marked as two-handed:

- Weapon 2's active item (if any) is moved to Inactive (Awaiting Sale).
- Weapon 2 slot displays as locked/grayed out with a "2H" label.
- Removing or selling the two-handed weapon unlocks Weapon 2.

**Implementation note:** The item paste format does not explicitly flag two-handed weapons. The add-item dialog includes a "Two-Handed" checkbox that the user toggles manually. This checkbox appears for both the main Weapon 1 and AG Weapon 1 slots.

### 3.5 Animate Guardian Slots

Each character has an optional **Animate Guardian** section, toggled by a button in the gear view header. When enabled, a separate gear layout is shown below the main grid with the following slots:

| Slot Key      | Label       |
|---------------|-------------|
| ag_weapon1    | Weapon 1    |
| ag_weapon2    | Weapon 2    |
| ag_helmet     | Helmet      |
| ag_bodyArmour | Body Armour |
| ag_gloves     | Gloves      |
| ag_boots      | Boots       |

The AG toggle state (`agEnabled`) is stored per character in the save file. AG slots follow the same item lifecycle rules as main gear slots.

---

## 4. Views & Navigation

### 4.1 Top Navigation Bar

```
[ Gear ]  [ Reports ]  [ Awaiting Sale (3) ]     [ Save ]  [ Load ]
```

- **Gear** and **Reports** are the main views.
- **Awaiting Sale** shows the count badge and opens the awaiting sale panel/modal.
- **Save/Load** on the right side. An "unsaved changes" indicator appears after any modification.

### 4.2 Gear View (Default)

**Top bar:** League selector → Character selector (filtered to selected league) → `[+ Add League]` `[+ Add Character]`

**Main area:** Three-column layout:
- **Left column:** 10 Skill Gem slots (vertical)
- **Center:** Visual gear grid
- **Right column:** 10 Jewel slots (vertical)

Below the center grid:
- **[Animate Guardian]** toggle button — shows/hides the AG gear section
- **Show Slot History** checkbox — when enabled, each slot shows a collapsed list of previous items with purchase/sell prices
- **Total Current Gear Cost** summary line

**Slot tile display:**
- Empty slots: slot name in muted text, click to open Add Item modal.
- Occupied slots with image: large item image filling the tile, purchase price in small footer. Item name is **not** shown when an image is present.
- Occupied slots without image (fallback): item name, base type, and purchase price as text.
- Loading state: shimmer placeholder while image is being fetched.
- Hovering an occupied slot shows the full PoE-style tooltip (see Section 6).

**Awaiting Sale panel/modal:**
- Shows all Inactive (Awaiting Sale) items for the selected league.
- Each entry shows: item name, base type, last character, purchase price.
- Action: Enter a sell price → moves item to Sold.

### 4.3 Add/Edit Item (Modal)

Triggered by clicking an empty slot or "Replace" from the tooltip action buttons.

**Fields:**

1. **Item Paste** — Large textarea. User pastes the full item text block (see Section 5). Live preview updates as text is pasted.
2. **Two-Handed** — Checkbox (only shown for Weapon 1 and AG Weapon 1 slots).
3. **Purchase Price** — Numeric input + currency dropdown (Chaos / Divine / Found).

**On save:**

- Parse the pasted text to extract: Item Name, Base Type, Item Class, Rarity, and full raw text.
- Assign an internal UUID.
- Trigger wiki image fetch in the background (see Section 7).
- If slot was occupied, move the previous item to Inactive (Awaiting Sale).
- If Weapon 1 + Two-Handed, handle Weapon 2 deactivation.

### 4.4 Sell Item (Modal)

Triggered from the Awaiting Sale list or the "Sell" action button on the tooltip.

**Fields:**

1. **Sell Price** — Numeric input + currency dropdown (Chaos / Divine / Found).

**On save:**

- Item state → Inactive (Sold).
- If the item was active, slot becomes empty.

### 4.5 Reports View

**Top bar:** League selector. Divine:Chaos ratio input (numeric, e.g. 176). Ratio value is saved to the save file.

**Report content for selected league:**

#### Per-Character Table

| Column | Description |
|--------|-------------|
| Character Name | Character name |
| Current Gear Cost | Sum of active items' purchase prices (normalized to Divines) |
| Total Spent (All) | Sum of all items ever purchased for this character (active + inactive + sold), normalized to Divines |
| Total Sold | Sum of all sell prices for this character's sold items, normalized to Divines |
| Net Investment | Total Spent (All) − Total Sold |

#### League Totals Row

- **Total Current Gear Cost** — Sum across all characters.
- **Total Spent** — Sum across all characters.
- **Total Sold** — Sum across all characters.
- **Net Investment** — Total Spent − Total Sold.

#### Normalization Logic

```
If currency == "Divine": display_value = value
If currency == "Chaos":  display_value = value / ratio
If currency == "Found":  display_value = 0
```

---

## 5. Item Paste Format

Items are pasted as plain text blocks copied directly from the PoE in-game client (Ctrl+C on an item). Sections are separated by `--------` divider lines. Example:

```
Item Class: Body Armours
Rarity: Rare
Eagle Cloak
Saint's Hauberk
--------
Quality: +20% (augmented)
Armour: 628 (augmented)
Energy Shield: 98 (augmented)
--------
Requirements:
Level: 67
Str: 109
Int: 96
--------
Sockets: B-R-B-B-R
--------
Item Level: 80
--------
+46 to Armour
+102 to maximum Life
Regenerate 116.7 Life per second
+29% to Cold Resistance
Reflects 9 Physical Damage to Melee Attackers
+15% to Fire and Lightning Resistances (crafted)
```

### Parsing Rules

- **Line starting with "Item Class:"** → Extract item class (e.g. "Body Armours"). Store for reference.
- **Line starting with "Rarity:"** → Extract rarity (Normal / Magic / Rare / Unique). Used for tooltip border color.
- **First non-header line after Rarity** → Item Name (e.g. "Eagle Cloak").
- **Second non-header line after Rarity** → Base Type (e.g. "Saint's Hauberk"). For Normal rarity items, there may only be one line (the base type doubles as the name).
- **`--------` dividers** → Preserved for tooltip section rendering.
- **Lines ending with `(crafted)`** → Flagged as crafted mods for tooltip color styling.
- **Everything else** → Stored as raw text for tooltip display.

No Unique ID is present in the in-game copy format. Items are identified internally by a generated UUID.

### Rarity Values & Colors

| Rarity | Color |
|--------|-------|
| Normal | #C8C8C8 (white/gray) |
| Magic  | #8888FF (blue) |
| Rare   | #FFFF77 (yellow) |
| Unique | #AF6025 (orange-brown) |

---

## 6. Item Tooltip (PoE Style)

Shown by hovering over an occupied slot tile. Disappears when the mouse leaves the slot or tooltip (with a 120ms grace period to allow moving the cursor to the tooltip action buttons).

### Trigger & Positioning

- **Trigger:** `onmouseenter` on occupied slot tile.
- **Hide:** `onmouseleave` from slot tile schedules hide after 120ms. Hovering onto the tooltip itself cancels the timer. Leaving the tooltip reschedules the hide.
- **Position:** Tooltip appears to the right of the cursor; repositions left or up if it would overflow the viewport.

### Visual Style

- **Background:** Dark (#0c0c0e or similar), solid.
- **Border:** Color based on parsed Rarity.
- **Header:** Item Name (larger, rarity-colored), Base Type below it (smaller). Subtle rarity-tinted gradient background, separator line below.
- **Section dividers:** `--------` lines rendered as subtle horizontal rules.
- **Body text:** PoE-like font. Line coloring:
  - Property lines (Quality, Armour, etc.) → light gray.
  - Mod lines → cyan (#88ddcc).
  - Lines ending with `(crafted)` → blue (#8888FF), suffix stripped from display.
  - Flavour text sections (heuristic: italic passages at the end) → italic gold (#b09050).
- **Item image:** Shown at the **bottom** of the tooltip, centered, ~80px tall, below a separator. Only displayed when an image URL is available.
- **Purchase price:** Shown at the bottom, below a separator. e.g., "Purchased: 50 Chaos" or "Found".
- **Action buttons:** "Replace", "Sell", "Deactivate" — shown at the very bottom of the tooltip.

---

## 7. Wiki Image Integration

Item images are fetched from poewiki.net and displayed in slot tiles and the tooltip.

### 7.1 Search Term Logic

- Unique or Gem items → search by **item name**.
- All other rarities → search by **base type**.

### 7.2 Fetch Process (Two-Step)

1. **Cargo API query** — `https://www.poewiki.net/w/api.php?action=cargoquery&tables=items&fields=name,image&where=name%3D'<term>'&format=json&origin=*`
2. If a result with an `image` filename is returned, fetch the actual URL via **Imageinfo API** — `https://www.poewiki.net/w/api.php?action=query&titles=File:<filename>&prop=imageinfo&iiprop=url&format=json&origin=*`
3. Extract the direct image URL from the Imageinfo response.

### 7.3 Caching

- **Session cache:** `Map<searchTerm, url|null>` — avoids redundant API calls within a browser session.
- **Persistent cache:** `item.imageUrl` field on ItemObject — saved in the JSON file. On load, items with an existing `imageUrl` skip the fetch.
- Items awaiting fetch are marked `imageUrl: 'loading'` in memory. The `'loading'` value is normalized to `null` before saving.

### 7.4 Rate Limiting

- Maximum 1 request per second to poewiki.net (enforced via request queue).
- Failed fetches set `imageUrl` to `null`; no retry within the session.

### 7.5 UI States

| State | Tile Display |
|-------|-------------|
| `imageUrl === 'loading'` | Shimmer/pulse placeholder animation |
| `imageUrl` is a URL | Image fills tile; price shown in footer |
| `imageUrl === null` | Text fallback: name + base type + price |

---

## 8. Data Model

### 8.1 Save File Structure (JSON, version 2)

```json
{
  "version": 2,
  "divineRatio": 160,
  "leagues": [
    {
      "id": "uuid",
      "name": "3.28 Mirage",
      "characters": [
        {
          "id": "uuid",
          "name": "MikeFireWitch",
          "agEnabled": false,
          "slots": {
            "weapon1": {
              "activeItem": null,
              "isTwoHanded": false,
              "history": []
            },
            "weapon2": { "activeItem": null, "history": [] },
            "helmet": { "activeItem": null, "history": [] },
            "bodyArmour": { "activeItem": null, "history": [] },
            "gloves": { "activeItem": null, "history": [] },
            "boots": { "activeItem": null, "history": [] },
            "amulet": { "activeItem": null, "history": [] },
            "ring1": { "activeItem": null, "history": [] },
            "ring2": { "activeItem": null, "history": [] },
            "belt": { "activeItem": null, "history": [] },
            "flask1": { "activeItem": null, "history": [] },
            "flask2": { "activeItem": null, "history": [] },
            "flask3": { "activeItem": null, "history": [] },
            "flask4": { "activeItem": null, "history": [] },
            "flask5": { "activeItem": null, "history": [] },
            "gem1":  { "activeItem": null, "history": [] },
            "...": "gem2 through gem10",
            "jewel1": { "activeItem": null, "history": [] },
            "...": "jewel2 through jewel10",
            "ag_weapon1": { "activeItem": null, "isTwoHanded": false, "history": [] },
            "ag_weapon2": { "activeItem": null, "history": [] },
            "ag_helmet": { "activeItem": null, "history": [] },
            "ag_bodyArmour": { "activeItem": null, "history": [] },
            "ag_gloves": { "activeItem": null, "history": [] },
            "ag_boots": { "activeItem": null, "history": [] }
          }
        }
      ],
      "awaitingSale": []
    }
  ]
}
```

**Version history:**
- `version: 1` — original format, no `imageUrl` on items, no `divineRatio` at root.
- `version: 2` — adds `imageUrl` to ItemObject, `divineRatio` at root, gem/jewel/AG slots, `agEnabled` per character. v1 files are automatically migrated on load.

### 8.2 ItemObject

```json
{
  "id": "internal-uuid",
  "name": "Eagle Cloak",
  "baseType": "Saint's Hauberk",
  "itemClass": "Body Armours",
  "rarity": "Rare",
  "rawText": "full pasted text (including dividers)",
  "imageUrl": "https://..." ,
  "purchasePrice": {
    "value": 50,
    "currency": "chaos"
  },
  "sellPrice": null,
  "status": "active",
  "dateAdded": "ISO timestamp",
  "dateSold": null
}
```

`imageUrl` may be: a full HTTPS URL (cached), `null` (not found or not yet fetched), or `'loading'` (in-memory only, never written to disk).

### 8.3 AwaitingSaleObject

```json
{
  "item": "ItemObject",
  "lastCharacterId": "uuid",
  "lastCharacterName": "MikeFireWitch",
  "lastSlot": "helmet"
}
```

Items in the awaiting sale list have `status: "awaiting_sale"` and retain their `lastCharacterId` for reporting attribution.

---

## 9. Save/Load

### 9.1 Save

- **Trigger:** Manual "Save" button in the top nav.
- **Method:** Serialize the full data model to JSON, trigger a browser download of the `.json` file.
- **Normalization:** `imageUrl: 'loading'` values are written as `null` to avoid stale loading states across sessions.
- **Filename:** `poe-gear-tracker.json`.

### 9.2 Load

- **Trigger:** Manual "Load" button in the top nav.
- **Method:** File input dialog, user selects a `.json` file, parse and load into memory.
- **Migration:** v1 files are detected and migrated to v2 in-memory (adds `imageUrl: null` to all items, initializes gem/jewel/AG slots, sets `divineRatio: 160`).

### 9.3 Unsaved Changes Indicator

No auto-save (no localStorage). A subtle indicator is shown in the nav after any modification, reminding the user to save.

---

## 10. UI Layout

### 10.1 Top Navigation Bar

```
[ Gear ]  [ Reports ]  [ Awaiting Sale (3) ]  ● unsaved     [ Save ]  [ Load ]
```

### 10.2 Gear View Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  League: [3.28 Mirage ▼]   Char: [FireWitch ▼]                   │
│  [+ Add League]  [+ Add Character]                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────┐  ┌──────────────────────────────┐  ┌──────┐            │
│  │ Gem1 │  │      . Helmet  Amulet . .    │  │ J1   │            │
│  │ Gem2 │  │  W1  R1  Body  Body  R2  W2  │  │ J2   │            │
│  │ ...  │  │  W1  Gl  Belt  Belt  Bt  W2  │  │ ...  │            │
│  │ Gem10│  │      F1  F2    F3    F4  F5  │  │ J10  │            │
│  └──────┘  └──────────────────────────────┘  └──────┘            │
│                                                                    │
│  [Animate Guardian ▼]                                             │
│  (AG gear grid shown here when enabled)                           │
│                                                                    │
│  ☐ Show Slot History                                              │
│  Total Current Gear: 14.3 Divine                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.3 Reports View Layout

```
┌─────────────────────────────────────────────────┐
│  League: [3.28 Mirage ▼]   1 Divine = [176] Chaos │
├─────────────────────────────────────────────────┤
│                                                    │
│  Character      Current  Total Spent  Sold   Net   │
│  ─────────────────────────────────────────────── │
│  FireWitch       14.3d     22.1d      6.5d  15.6d │
│  IceShadow        8.7d     15.3d      4.2d  11.1d │
│  ─────────────────────────────────────────────── │
│  League Total    23.0d     37.4d     10.7d  26.7d │
│                                                    │
└─────────────────────────────────────────────────┘
```

---

## 11. Frontend Implementation

### 11.1 Technology

- **Single HTML file** with embedded CSS and JS. No build tools, no frameworks, no localStorage.
- Vanilla JavaScript with modern ES6+ syntax.
- CSS Grid for the gear layout, Flexbox for the nav and panels.

### 11.2 Styling

- Dark theme to match PoE aesthetic (dark grays, muted golds, blacks).
- Font: `"FontinSmallCaps"` with fallback to `"Fontin"`, `Georgia`, `serif` for item text; clean sans-serif for UI chrome.
- Slot tiles: image-dominant with a small price footer. Min-height ~72px for main slots; smaller for gem/jewel/flask slots.
- Tooltips styled per Section 6.

### 11.3 Key Interactions

| User Action | Result |
|-------------|--------|
| Click empty slot | Opens Add Item modal |
| Hover occupied slot | Shows PoE-style tooltip |
| Tooltip → "Replace" | Opens Add Item modal for that slot |
| Tooltip → "Sell" | Opens Sell Price modal |
| Tooltip → "Deactivate" | Moves item to Awaiting Sale, no sell price |
| Click "Awaiting Sale" nav badge | Opens Awaiting Sale panel |
| Awaiting Sale → "Set Sell Price" | Opens Sell Price modal |
| Click "Animate Guardian" button | Toggles AG section visibility |

### 11.4 Slot Definition Constants

```js
const MAIN_SLOTS  = [ {key:'weapon1'}, {key:'weapon2'}, {key:'helmet'}, ... ]; // 15 slots
const GEM_SLOTS   = Array.from({length:10}, (_,i) => ({key:`gem${i+1}`,   lbl:`Gem ${i+1}`}));
const JEWEL_SLOTS = Array.from({length:10}, (_,i) => ({key:`jewel${i+1}`, lbl:`Jewel ${i+1}`}));
const AG_SLOTS    = [
  {key:'ag_weapon1', lbl:'Weapon 1'},
  {key:'ag_weapon2', lbl:'Weapon 2'},
  {key:'ag_helmet',  lbl:'Helmet'},
  {key:'ag_bodyArmour', lbl:'Body'},
  {key:'ag_gloves',  lbl:'Gloves'},
  {key:'ag_boots',   lbl:'Boots'},
];
const ALL_SLOTS = [...MAIN_SLOTS, ...GEM_SLOTS, ...JEWEL_SLOTS, ...AG_SLOTS];
```

---

## 12. Future Enhancements

1. **Path of Building format support** — Support the PoB paste format (`{crafted}` prefix, `Implicits: N`, `Unique ID:`) as an alternate input.
2. **Import from PoE API** — Pull character gear directly from the official PoE API.
3. **Multiple currencies** — Exalted Orbs, Mirrors, etc.
4. **Chart/graph in reports** — Visual spending over time per league.
5. **Drag-and-drop reordering** — Move items between characters.
6. **Bulk item paste** — Parse multiple items from a Path of Building export.
