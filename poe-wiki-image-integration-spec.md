# PoE Gear Cost Tracker — Wiki Image Integration Spec

## 1. Overview

This document specifies how the Gear Cost Tracker will fetch and display item images (equipment, skill gems, jewels) using the community wiki's Cargo database and MediaWiki API. Images are fetched on demand when an item is added and cached in the save file to minimize repeat requests.

---

## 2. Target Wiki

All image lookups use the Path of Exile community wiki:

| Wiki | Base URL |
|------|----------|
| poewiki.net | `https://www.poewiki.net` |

The base URL is hardcoded. No per-league wiki setting is needed.

---

## 3. Wiki API Endpoints

All requests are standard HTTP GET with `format=json` and `origin=*` (for CORS). No authentication is required.

### 3.1 Cargo Query — Look Up Item by Base Type

This queries the wiki's `items` Cargo table to find the inventory icon filename for a given base type or item name.

**Endpoint:**
```
https://www.poewiki.net/w/api.php?action=cargoquery
  &tables=items
  &fields=items.name,items.inventory_icon
  &where=items.name="{searchTerm}"
  &limit=1
  &format=json
  &origin=*
```

**Search Strategy (in order):**

1. **By base type** — e.g. `items.name="Saint's Hauberk"`. This is the most reliable match for rare/magic/normal items.
2. **By item name** — e.g. `items.name="Shavronne's Wrappings"`. Used for unique items where the name is distinct from the base type.
3. **Fallback with LIKE** — e.g. `items.name LIKE "%Tornado Wand%"`. Last resort if exact match fails.

**Response Example:**
```json
{
  "cargoquery": [
    {
      "title": {
        "name": "Saint's Hauberk",
        "inventory icon": "Saint's Hauberk inventory icon.png"
      }
    }
  ]
}
```

The `inventory icon` field (note: Cargo returns field names with spaces, not underscores) gives us the filename to use in the next step.

### 3.2 MediaWiki Imageinfo — Get the Actual Image URL

Once we have the icon filename, we fetch the hosted image URL.

**Endpoint:**
```
https://www.poewiki.net/w/api.php?action=query
  &titles=File:{iconFilename}
  &prop=imageinfo
  &iiprop=url
  &format=json
  &origin=*
```

**Example:**
```
https://www.poewiki.net/w/api.php?action=query
  &titles=File:Saint's Hauberk inventory icon.png
  &prop=imageinfo
  &iiprop=url
  &format=json
  &origin=*
```

**Response Example:**
```json
{
  "query": {
    "pages": {
      "12345": {
        "title": "File:Saint's Hauberk inventory icon.png",
        "imageinfo": [
          {
            "url": "https://www.poewiki.net/images/a/ab/Saint%27s_Hauberk_inventory_icon.png"
          }
        ]
      }
    }
  }
}
```

Extract `imageinfo[0].url` from the first (and only) page result. If `pages` contains a key with a negative ID (e.g. `-1`), the image doesn't exist on the wiki.

### 3.3 Constructed URL Shortcut

If we already know or can infer the icon filename follows the standard wiki naming convention (`{ItemName} inventory icon.png`), we can skip the Cargo query (Step 3.1) and go directly to the Imageinfo endpoint (Step 3.2).

This is useful because the wiki's naming convention is consistent:

- Base types: `Saint's Hauberk inventory icon.png`
- Uniques: `Shavronne's Wrappings inventory icon.png`
- Skill gems: `Fireball inventory icon.png`
- Jewels: `Cobalt Jewel inventory icon.png`

However, the Cargo query remains the reliable fallback when the naming convention doesn't match (e.g., alternate art, items with special characters, or renamed items).

---

## 4. Item Type Lookup Strategies

Different item types require different lookup approaches based on what we parse from the pasted item text.

### 4.1 Equipment (Armor, Weapons, Accessories, Flasks)

**For Rare / Magic / Normal items:**
- Search by **base type** (line 2 of the parsed header, e.g., "Saint's Hauberk").
- The base type image is the correct visual since rare/magic items share their base type's 2D art.

**For Unique items:**
- Search by **item name** (line 1, e.g., "Shavronne's Wrappings").
- Uniques have their own distinct 2D art tied to their name, not the base type.

**Detection:** Use the parsed `rarity` field.
```
if rarity === "Unique" → search by item name
else → search by base type
```

### 4.2 Skill Gems

Skill gems paste like:
```
Item Class: Skill Gems
Rarity: Gem
Fireball
--------
...
```

- The item name IS the gem name (e.g., "Fireball").
- Search by item name directly.
- Wiki filename: `Fireball inventory icon.png`

**Support gems** work the same way: `Fireball of Conflagration`, `Added Fire Damage Support`, etc.

### 4.3 Jewels

Jewels paste like:
```
Item Class: Jewels
Rarity: Rare
Brood Star
Cobalt Jewel
--------
...
```

**For Rare / Magic jewels:**
- Search by **base type** (e.g., "Cobalt Jewel").
- All rare Cobalt Jewels share the same icon.

**For Unique jewels:**
- Search by **item name** (e.g., "Militant Faith").

### 4.4 Summary Table

| Item Type | Rarity | Search Term | Wiki Field |
|-----------|--------|-------------|------------|
| Equipment | Normal/Magic/Rare | Base Type | `items.name` |
| Equipment | Unique | Item Name | `items.name` |
| Skill Gem | Gem | Item Name | `items.name` |
| Jewel | Normal/Magic/Rare | Base Type | `items.name` |
| Jewel | Unique | Item Name | `items.name` |
| Flask | Normal/Magic/Rare | Base Type | `items.name` |
| Flask | Unique | Item Name | `items.name` |

---

## 5. Image Caching Strategy

To avoid repeated API calls, image URLs are cached at two levels.

### 5.1 Session Cache (In-Memory)

A JavaScript `Map` keyed by the search term that stores the resolved image URL. This prevents duplicate fetches during a single session when the same base type appears on multiple characters.

```javascript
const imageCache = new Map();
// Key: "Saint's Hauberk" → Value: "https://www.poewiki.net/images/..."
```

### 5.2 Persistent Cache (Save File)

Store the resolved image URL on the `ItemObject` itself:

```json
{
  "id": "internal-uuid",
  "name": "Eagle Cloak",
  "baseType": "Saint's Hauberk",
  "itemClass": "Body Armours",
  "rarity": "Rare",
  "rawText": "...",
  "imageUrl": "https://www.poewiki.net/images/a/ab/Saint%27s_Hauberk_inventory_icon.png",
  "purchasePrice": { ... },
  ...
}
```

**On item add:**
1. Check session cache for the search term.
2. If miss, call the wiki API (Cargo → Imageinfo).
3. Store the resulting URL in both the session cache and the item's `imageUrl` field.
4. If the API call fails or returns no result, set `imageUrl` to `null` (the UI falls back to text-only display).

**On load from save file:**
- Items already have `imageUrl` populated. No API calls needed.
- Populate the session cache from loaded items to avoid redundant fetches if new items share base types.

### 5.3 Manual Refresh

Provide a small refresh button on the tooltip or item detail that re-fetches the image URL from the wiki. This handles cases where:
- The wiki added an image after the item was first saved.
- The cached URL is broken (wiki moved images).

---

## 6. API Client Implementation

### 6.1 Core Fetch Function

```javascript
const WIKI_BASE = 'https://www.poewiki.net';

async function fetchItemImage(searchTerm) {
  if (imageCache.has(searchTerm)) return imageCache.get(searchTerm);

  let iconFilename = null;

  // Strategy 1: Try constructed filename directly
  const guessedFilename = `${searchTerm} inventory icon.png`;
  const directUrl = await fetchImageUrl(guessedFilename);
  if (directUrl) {
    imageCache.set(searchTerm, directUrl);
    return directUrl;
  }

  // Strategy 2: Cargo query fallback
  iconFilename = await cargoLookup(searchTerm);
  if (iconFilename) {
    const url = await fetchImageUrl(iconFilename);
    if (url) {
      imageCache.set(searchTerm, url);
      return url;
    }
  }

  // No image found
  imageCache.set(searchTerm, null);
  return null;
}
```

### 6.2 Cargo Query

```javascript
async function cargoLookup(searchTerm) {
  const params = new URLSearchParams({
    action: 'cargoquery',
    tables: 'items',
    fields: 'items.name,items.inventory_icon',
    where: `items.name="${searchTerm}"`,
    limit: '1',
    format: 'json',
    origin: '*',
  });

  try {
    const resp = await fetch(`${WIKI_BASE}/w/api.php?${params}`);
    const data = await resp.json();
    const result = data?.cargoquery?.[0]?.title;
    return result?.['inventory icon'] || null;
  } catch (e) {
    console.warn('Cargo lookup failed:', e);
    return null;
  }
}
```

### 6.3 Imageinfo Query

```javascript
async function fetchImageUrl(filename) {
  const params = new URLSearchParams({
    action: 'query',
    titles: `File:${filename}`,
    prop: 'imageinfo',
    iiprop: 'url',
    format: 'json',
    origin: '*',
  });

  try {
    const resp = await fetch(`${WIKI_BASE}/w/api.php?${params}`);
    const data = await resp.json();
    const pages = data?.query?.pages;
    if (!pages) return null;
    const page = Object.values(pages)[0];
    // Negative page ID means the file doesn't exist
    if (page?.imageinfo?.[0]?.url) return page.imageinfo[0].url;
    return null;
  } catch (e) {
    console.warn('Imageinfo lookup failed:', e);
    return null;
  }
}
```

### 6.4 Determining the Search Term

Called during item add, after parsing:

```javascript
function getImageSearchTerm(parsedItem) {
  // Unique items and gems use the item name
  if (parsedItem.rarity === 'Unique' || parsedItem.rarity === 'Gem') {
    return parsedItem.name;
  }
  // Everything else uses the base type
  return parsedItem.baseType;
}
```

---

## 7. UI Integration

### 7.1 Gear Grid Slot Tiles

When a slot has an item with a valid `imageUrl`, show the image as a small icon within the slot tile.

```
┌────────────────────┐
│ HELMET             │  ← slot label
│ [img] Eagle Cloak  │  ← 32×32 icon + item name
│       Saint's Haub │  ← base type
│              50c   │  ← price
└────────────────────┘
```

- Image size: **32×32px** in the main gear grid, **24×24px** in gem/jewel columns.
- If `imageUrl` is null, display text-only (current behavior).
- Use `object-fit: contain` to handle varying icon aspect ratios.
- Add a subtle border matching the rarity color around the image.

### 7.2 Tooltip

Show the item image larger in the tooltip header, beside the item name.

```
┌─────────────────────────────────────┐
│  [64×64 img]  Eagle Cloak          │
│               Saint's Hauberk      │
├─────────────────────────────────────┤
│  Quality: +20% (augmented)         │
│  Armour: 628 (augmented)           │
│  ...                               │
├─────────────────────────────────────┤
│  Purchased: 50 Chaos               │
├─────────────────────────────────────┤
│  [Replace] [Awaiting Sale] [Sell]  │
└─────────────────────────────────────┘
```

- Image size: **64×64px** in the tooltip.
- Positioned to the left of the item name/base type in the header area.
- If no image, the header layout stays the same (text-only, centered as it is now).

### 7.3 Awaiting Sale List

Show a small **24×24px** icon next to the item name in each awaiting sale row.

### 7.4 Loading State

While the image is being fetched (during item add):
- Show a subtle loading spinner or pulsing placeholder in the slot.
- The item is added immediately with `imageUrl: null`, then updated asynchronously when the fetch completes.
- On fetch completion, re-render the affected slot tile.

### 7.5 Offline / Failure Handling

- If the wiki is unreachable, the image fetch silently fails and `imageUrl` stays `null`.
- No error toast for image fetch failures (they're non-critical).
- Items without images display exactly as they do now (text-only).
- Console warnings are logged for debugging.

---

## 8. Rate Limiting & Etiquette

The MediaWiki API documentation requests that tools using their API be polite:

- **No parallel batch requests.** Fetch images one at a time, sequentially.
- **Reasonable rate.** No more than ~1 request per second when doing bulk operations.
- **Cache aggressively.** Once an image URL is resolved, don't re-fetch unless the user explicitly requests a refresh.
- **User-Agent header.** While browsers may not allow setting this on fetch, if possible, include a descriptive header: `PoE-Gear-Tracker/1.0 (personal use tool)`.

### 8.1 Batch Loading on Save File Import

When loading a save file, items already have `imageUrl` cached. No API calls are needed. If a future "refresh all images" feature is added, it should throttle requests to ~1/second with a progress indicator.

---

## 9. Data Model Changes

### 9.1 ItemObject (Updated)

Add `imageUrl` field:

```json
{
  "id": "internal-uuid",
  "name": "Eagle Cloak",
  "baseType": "Saint's Hauberk",
  "itemClass": "Body Armours",
  "rarity": "Rare",
  "rawText": "...",
  "imageUrl": "https://www.poewiki.net/images/a/ab/...",
  "purchasePrice": { ... },
  "sellPrice": null,
  "status": "active",
  "dateAdded": "...",
  "dateSold": null
}
```

### 9.2 Save File Version

Bump to `version: 2` if needed. The loader should handle v1 saves gracefully by setting `imageUrl: null` on all existing items.

---

## 10. Integration Points in Existing Code

### 10.1 Item Add Flow (`submitAdd`)

After `parseItem()` succeeds:

```javascript
// Existing: parse item, set price, equip in slot
const p = parseItem(text);
p.purchasePrice = ...;
slot.activeItem = p;

// New: kick off async image fetch
const searchTerm = getImageSearchTerm(p);
fetchItemImage(searchTerm).then(url => {
  p.imageUrl = url;
  renderGear(); // Re-render to show the image
  markDirty();  // URL is part of saved state
});
```

### 10.2 Gear Render (`buildSlotTile`)

Add image element before the item name:

```javascript
if (item.imageUrl) {
  const imgSize = isGemOrJewelSlot ? 24 : 32;
  inner += `<img src="${h(item.imageUrl)}" 
    width="${imgSize}" height="${imgSize}" 
    style="object-fit:contain;border:1px solid ${rColor(item.rarity)}30;border-radius:2px;float:left;margin-right:6px"
    onerror="this.style.display='none'">`;
}
```

### 10.3 Tooltip Render (`showTip`)

Add image to tooltip header:

```javascript
if (item.imageUrl) {
  tipHead.innerHTML = `
    <div style="display:flex;align-items:center;gap:12px;justify-content:center">
      <img src="${h(item.imageUrl)}" width="64" height="64" 
        style="object-fit:contain" onerror="this.style.display='none'">
      <div>
        <div class="tip-name" style="color:${col}">${h(item.name)}</div>
        <div class="tip-base" style="color:${col}">${...}</div>
      </div>
    </div>`;
}
```

---

## 11. Error Handling & Edge Cases

| Scenario | Handling |
|----------|----------|
| Wiki is down / unreachable | `imageUrl` set to `null`, text-only display, console warning |
| Item not found in Cargo DB | Try direct filename guess → if both fail, `null` |
| Image file exists in Cargo but not uploaded | Imageinfo returns negative page ID → `null` |
| Special characters in item name (apostrophes, etc.) | URL-encode the search term in API params; `URLSearchParams` handles this |
| Same base type across multiple items | Session cache returns the same URL instantly |
| User loads a v1 save file | Migration: set `imageUrl: null` on all items |
| CORS blocked | poewiki.net supports `origin=*`; shouldn't occur. If it does, silent failure |
| Very long item names | API handles arbitrary query lengths; no truncation needed |

---

## 12. Future Considerations

1. **Batch image refresh** — "Refresh All Images" button that re-fetches URLs for all items across all leagues, throttled to 1 req/sec, with a progress bar.
2. **Thumbnail sizing** — The Imageinfo API supports `iiurlwidth` parameter to request server-side thumbnails at specific widths, reducing bandwidth.
3. **Fallback to poecdn** — If the wiki image is missing, attempt to construct a `web.poecdn.com` URL as a secondary source (requires maintaining a base-type-to-art-path mapping, which is complex).
4. **Image preloading** — When switching characters, preload images for all equipped items to avoid flicker.
