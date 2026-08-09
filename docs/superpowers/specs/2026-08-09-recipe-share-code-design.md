# Recipe share code (encoded export/import) — design

## Problem

`exportData()`/`importData()` back up the *entire* app (settings + pantry + packaging + recipes) as a downloadable `.json` file. That's the right tool for backing up your own device, but it's a bad fit for "send my friend this one brownie recipe" — too much data, and it requires a file-transfer step instead of a copy-paste into a chat app.

This adds a second, separate feature: pick one or more recipes, get back a short text code, paste it to a friend, they paste it into their own app to import. The existing whole-app JSON export/import is untouched.

## Core problem: ingredients and packaging don't map across users

Recipes reference ingredients and packaging by ID, pointing into *this device's* pantry/packaging warehouse. Those IDs are meaningless on a friend's device — they don't have "the same butter" at the same ID, or possibly at all.

Resolution: share ingredients and packaging **by value, not by reference**. At share time, each ingredient/packaging usage is snapshotted (name + the unit cost it resolves to right now). On import, the recipe carries these snapshots as **unlinked** entries — the recipe works immediately (cost calculates from the snapshot), but nothing points at the receiver's pantry yet. A lightweight in-app flow then lets the receiver map each unlinked entry to an existing pantry/packaging item, or add it as a new one, at their own pace.

## Payload format

```json
{
  "v": 1,
  "type": "recipe-bundle",
  "packaging": [
    { "key": "pk0", "name": "แก้วกาแฟ", "unitCost": 5 }
  ],
  "recipes": [
    {
      "name": "บราวนี่", "category": "ขนม", "yieldQty": 10, "yieldUnit": "ชิ้น",
      "laborRate": 60, "laborHours": 1, "gasPct": 5, "deliveryCost": 0,
      "overheadPct": 20, "targetPct": 40,
      "packagingKey": "pk0",
      "ingredients": [
        { "name": "เนยเค็ม", "qty": 200, "unit": "กรัม", "unitCost": 0.18 }
      ]
    }
  ]
}
```

Notes:
- A single-recipe share is a bundle with one `recipes` entry — same code path as multi-select, no special case.
- `packaging` is deduped once per bundle; recipes reference it by a bundle-local `key` (not a receiver-side ID), so several selected recipes sharing the same packaging item don't repeat it in the payload.
- `packagingKey` is omitted/`null` when the recipe uses no packaging. This is a real, common case — a dish served directly (on a plate, no takeaway container) already has "— ไม่ใช้บรรจุภัณฑ์ —" as a first-class option in the app today (`packagingId:''`), and it must round-trip through share/import as cleanly as any other field: no packaging in, no packaging out, no unlinked-packaging banner, no phantom mapping prompt. It is a distinct third state from "has packaging" (`packagingKey` set) and "has packaging but unmapped" (see below) — never conflate "no packaging" with "unmapped packaging."
- **Excluded fields:** `sellPrice` (a business's live selling price shouldn't silently overwrite the receiver's own pricing — resets to `0` on import) and `note` (currently has no UI anywhere in the app; not worth carrying).
- `v` is a format version. Import rejects payloads with `v` greater than what this app version understands, with a message asking the user to update the app, rather than silently mis-importing.

## Encoding

`JSON.stringify(payload)` → compress with a small hand-written LZ-style dictionary compressor (same algorithmic idea as the classic "lz-string" approach: back-reference repeated substrings via a growing dictionary; effective on repetitive JSON keys and repeated Thai ingredient/unit strings) → encode to a **URL-safe base64** alphabet (`-`/`_` instead of `+`/`/`, no padding) → prefix with `FCC1:` (Food Cost Calculator, format v1).

The `FCC1:` prefix lets import cheaply reject non-matching paste content (wrong app, garbage, accidental partial paste) before attempting to decode, and gives room for a `FCC2:` etc. prefix if the format changes incompatibly later.

The compressor is implemented inline in `index.html` (no CDN dependency), consistent with the app's existing single-file, no-external-fetch design (same spirit as the embedded Tangmo font).

## Data model changes

**Ingredients** — no new mode plumbing needed. `ingUnitCost`/`ingUnitLabel`/`ingName`/`ingCost` already fall back to reading `unitCost`/`unit`/`name` directly off the ingredient for any `mode !== 'pantry'` (this generic branch predates the pantry-only ingredient change and was never removed). Imported ingredients use `mode: 'unlinked'`:

```js
{ id, mode: 'unlinked', name, qty, unit, unitCost }
```

**Packaging** — a recipe holds a single `packagingId`; add one parallel optional field:

```js
r.unlinkedPackaging = { name, unitCost } | null
```

`calcRecipe` change: when `packagingId` is empty/null but `unlinkedPackaging` is set, use `unlinkedPackaging.unitCost` as the packaging unit cost. Once mapped or added to the warehouse, `packagingId` is set and `unlinkedPackaging` is cleared — from then on the recipe behaves exactly like any non-imported recipe.

Importing a recipe whose `packagingKey` was `null` (dish served directly, no packaging) must produce `packagingId:'' , unlinkedPackaging:null` — the same shape a locally-created "no packaging" recipe already has. It is not an unlinked/pending state and must not trip `recipeNeedsMapping` below.

**"Needs mapping" check** (drives the badge/banner):

```js
function recipeNeedsMapping(r){
  return r.ingredients.some(i => i.mode !== 'pantry') || !!r.unlinkedPackaging;
}
```

## Mapping UI

Reuses the existing dropdowns rather than adding new modes/buttons wherever possible:

- **Ingredient row:** when `mode==='unlinked'`, the pantry `<select>` gets one extra option pinned at the top and pre-selected: `🟡 (นำเข้า) เนยเค็ม — 0.18 บาท/กรัม`. Picking any real pantry item from that same dropdown converts the row to `mode:'pantry'` with that `pantryId` — this *is* the "map to existing" action, no separate control needed. A small "+ เพิ่มเข้าคลัง" link next to the row creates a new pantry item pre-filled from the snapshot (`buyQty:1`, `buyPrice:unitCost`, unit family guessed from the snapshot's unit string — กรัม/กก.→mass, มล./ลิตร→volume, ฟอง→egg, anything else (e.g. a sender's free-text unit like "ถุง") falls back to piece/ชิ้น as a rough default) and immediately links the row to it. The receiver can always correct the created item's category/unit/price afterward via the normal pantry edit form — this default just needs to get them to a working starting point, not a perfect one.
- **Packaging field:** same pattern on the "บรรจุภัณฑ์ที่ใช้" dropdown — a pinned `🟡 (นำเข้า) ...` option, plus a "+ เพิ่มเข้าคลังบรรจุภัณฑ์" link that creates a new packaging item from the snapshot (`buyQty:1`, `buyUnit:'ชิ้น'`, `piecesPerBox:1`, `buyPrice:unitCost`) and links it.

Both newly-created items are ordinary, fully-editable pantry/packaging entries afterward — nothing special persists once mapped.

## Flagging unmapped recipes

- Recipe list cards: a small badge (e.g. "ต้องเชื่อมวัตถุดิบ") shown when `recipeNeedsMapping(r)` is true.
- Recipe editor: a banner at the top listing which ingredients/packaging are still unmapped. Not manually dismissible — it disappears automatically once everything is mapped, since a dismissed-but-still-broken recipe would be confusing.

## Share & import flows

- **Single-recipe share:** a "แชร์" button on each recipe card (alongside the existing ทำสำเนา/ลบ) encodes just that recipe and opens a modal with a read-only textarea containing the code plus a "คัดลอก" button.
- **Multi-select:** hold-tap (long-press) a recipe card enters selection mode. The list toolbar swaps to a selection bar — "เลือกแล้ว N สูตร" with "คัดลอกโค้ด" and "ยกเลิก" — and tapping other cards toggles a checkbox overlay. "คัดลอกโค้ด" opens the same share modal with the bundle of selected recipes. (Desktop gets the same long-press gesture via pointer events — mouse-down-and-hold — no separate toolbar entry point for now, per earlier decision.)
- **Import:** a "นำเข้าโค้ด" button in the recipe list toolbar opens a modal with a paste textarea and a "นำเข้า" button.
  - Success: recipe(s) are prepended to `state.recipes` with freshly generated IDs (never overwrites existing recipes, even re-importing the same code creates a new copy — same duplication behavior as "ทำสำเนา"), modal closes.
  - Failure: inline error in the modal — "โค้ดไม่ถูกต้องหรือเสียหาย" for anything that doesn't start with `FCC1:` or fails to decode/parse/validate against the expected shape; a distinct "โค้ดนี้สร้างจากแอปเวอร์ชันใหม่กว่า กรุณาอัปเดตแอป" message if `v` is newer than supported.
- **Copy mechanism:** try `navigator.clipboard.writeText`; if unavailable (common when the app is opened via `file://`, where the Clipboard API is often blocked) fall back to `document.execCommand('copy')` on the textarea; if that also fails, auto-select the textarea's text and show a "แตะเพื่อเลือกแล้วคัดลอก" hint so the user can copy manually.

## Out of scope

- Any server/network component — this stays a purely local, copy-paste mechanism.
- Bulk actions beyond "share selected" (bulk delete/duplicate) — selection mode exists only to drive the share flow.
- Editing the whole-app JSON export/import — unchanged, stays as the full-backup mechanism.
- A desktop-specific (non-long-press) entry point into selection mode — deferred per earlier decision.
