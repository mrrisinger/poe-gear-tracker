# PoE Gear Cost Tracker

A single-file HTML tool for tracking gear costs across Path of Exile leagues and characters. No server, no install — just open the file in a browser.

## Features

- Visual gear grid matching the in-game equipment layout
- Skill Gem slots (10) and Jewel slots (10)
- Animate Guardian gear section per character
- Automatic item images fetched from the PoE wiki
- Purchase and sell price tracking per item
- Awaiting Sale list for items between characters
- Reports view with per-character and league-wide cost summaries
- JSON save/load — all data stays on your machine

## How to Use

### Adding an Item

1. **In Path of Exile**, hover over any item in your inventory or equipment screen and press **Ctrl+C** to copy the item data to your clipboard.
2. In the tracker, select your league and character, then click an empty gear slot.
3. Paste the copied text into the Item Paste field (**Ctrl+V**).
4. Enter the purchase price and click Save.

The tracker will automatically parse the item name, base type, rarity, and stats, and fetch the item image from the PoE wiki.

### Selling / Replacing an Item

- **Hover** over an occupied slot to see the item tooltip.
- Use the **Replace** button to swap in a new item (the old one moves to Awaiting Sale).
- Use the **Sell** button to record a sell price.
- Use the **Deactivate** button to move an item to Awaiting Sale without a sell price.

### Reports

Switch to the **Reports** tab to see per-character and league-wide spending summaries. Set the Divine:Chaos ratio to normalize all prices to Divines.

### Saving Your Data

Click **Save** in the top nav to download your data as a `.json` file. Click **Load** to restore it. There is no auto-save — save often.

## Supported Slots

| Section | Slots |
|---------|-------|
| Main Gear | Weapon 1, Weapon 2, Helmet, Body Armour, Gloves, Boots, Amulet, Ring 1, Ring 2, Belt, Flasks 1–5 |
| Skill Gems | Gem 1–10 |
| Jewels | Jewel 1–10 |
| Animate Guardian | Weapon 1, Weapon 2, Helmet, Body Armour, Gloves, Boots |

## Notes

- Two-handed weapons can be flagged with the Two-Handed checkbox when adding to Weapon 1, which locks the Weapon 2 slot.
- Item images are sourced from [poewiki.net](https://www.poewiki.net) and cached in your save file after the first fetch.
- The Divine:Chaos ratio is saved with your data so it persists between sessions.
