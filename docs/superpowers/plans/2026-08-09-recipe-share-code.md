# Recipe Share Code Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user pick one or more recipes, get a short pasteable text code, and let a friend paste that code into their own copy of the app to import those recipes — without touching the existing whole-app JSON file backup.

**Architecture:** Ingredients/packaging are snapshotted by value (name + resolved unit cost) into the share payload, never by ID (a friend's pantry has different IDs, or no matching item at all). JSON payload → hand-written LZW dictionary compressor → varint-packed bytes → URL-safe base64 → `FCC1:` prefix. On import, snapshots become `mode:'unlinked'` ingredients (cost calculates immediately, off the snapshot) and an `r.unlinkedPackaging` field, with in-editor UI to map each one to the receiver's own pantry/packaging or add it fresh. Everything lives in the single `index.html` file — no build step, no new files, no external dependency, consistent with the rest of the app.

**Tech Stack:** Vanilla JS, inline in `index.html`. No frameworks, no package manager, no test runner — this codebase has none of those, so verification uses `node --check` for syntax and live-browser JS assertions (via the Browser tool) for behavior, which is how every prior feature in this repo has been verified.

## Global Constraints

- Single file: all changes go into `/Users/jukkraputpikunyam/Satang/Bunny/food-cost-calculator/index.html`. No new files, no CDN links, no npm dependency.
- Share payload excludes `sellPrice` (resets to `0` on import) and `note` (unused field, no UI anywhere).
- `packagingKey`/`unlinkedPackaging` being absent/`null` means "no packaging" — a first-class, common state (dish served directly) — and must **never** be treated as "needs mapping." Only a present-but-unlinked ingredient/packaging counts as needing mapping.
- Imported recipes are always **new** entries with freshly generated IDs — import never overwrites or dedupes against existing recipes, even re-importing the same code.
- Every UI-touching task ends with `node --check` on the extracted `<script>` content (see Task 1, Step 2 for the exact command) before the browser-verification steps.
- Reference: design spec at `docs/superpowers/specs/2026-08-09-recipe-share-code-design.md`.

---

## Task 1: LZW compressor primitives

**Files:**
- Modify: `index.html` — add a new section right after the `esc()` helper (currently ends at line 311, immediately before `function pantryUnitCost(p){` at line 313).

**Interfaces:**
- Produces: `lzwCompress(str) -> number[]`, `lzwDecompress(codes) -> string`, `varintEncode(codes) -> number[]` (byte values 0-255), `varintDecode(bytes) -> number[]`, `bytesToBase64Url(bytes) -> string`, `base64UrlToBytes(s) -> number[]`, `compressString(str) -> string`, `decompressString(code) -> string`. All pure, no DOM/state access.

- [ ] **Step 1: Add the codec primitives to `index.html`**

Insert this new block immediately after the line `function esc(s){ return String(s==null?'':s).replace(/[&<>"']/g, c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }` (line 311) and before `function pantryUnitCost(p){` (line 313):

```js
/* ============================== SHARE CODE: CODEC ============================== */
/* ===== SHARE-CODE-CODEC:START ===== */
// Minimal LZW dictionary compressor over UTF-16 code units, plus varint byte
// packing and URL-safe base64. Not a port of any library — same well-known
// idea (back-reference repeated substrings via a growing dictionary), written
// fresh so share codes stay short without adding an external dependency.
// Iterating by code unit (not by Unicode code point) means every symbol that
// can appear — including each half of a surrogate pair — is guaranteed to
// already be seeded in the dictionary, so there's no "unseen first symbol"
// edge case to special-case.

function lzwCompress(str){
  const dict = new Map();
  for(let i=0;i<65536;i++) dict.set(String.fromCharCode(i), i);
  let nextCode = 65536;
  let w = '';
  const codes = [];
  for(let i=0;i<str.length;i++){
    const ch = str[i];
    const wc = w + ch;
    if(dict.has(wc)){
      w = wc;
    } else {
      codes.push(dict.get(w));
      dict.set(wc, nextCode++);
      w = ch;
    }
  }
  if(w !== '') codes.push(dict.get(w));
  return codes;
}

function lzwDecompress(codes){
  if(codes.length === 0) return '';
  const dict = new Map();
  for(let i=0;i<65536;i++) dict.set(i, String.fromCharCode(i));
  let nextCode = 65536;
  let w = dict.get(codes[0]);
  let result = w;
  for(let i=1;i<codes.length;i++){
    const k = codes[i];
    let entry;
    if(dict.has(k)) entry = dict.get(k);
    else if(k === nextCode) entry = w + w[0];
    else throw new Error('corrupt LZW stream');
    result += entry;
    dict.set(nextCode++, w + entry[0]);
    w = entry;
  }
  return result;
}

// Plain-arithmetic (no bitwise ops) LEB128-style varint, so there's no 32-bit
// overflow risk even if a very large bundle pushes dictionary codes high.
function varintEncode(codes){
  const bytes = [];
  for(const code of codes){
    let n = code;
    while(true){
      const b = n % 128;
      n = Math.floor(n / 128);
      if(n !== 0){ bytes.push(b | 0x80); } else { bytes.push(b); break; }
    }
  }
  return bytes;
}
function varintDecode(bytes){
  const codes = [];
  let n = 0, mult = 1;
  for(const b of bytes){
    n += (b & 0x7f) * mult;
    if(b & 0x80){ mult *= 128; } else { codes.push(n); n = 0; mult = 1; }
  }
  return codes;
}

const B64URL_CHARS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_';
function bytesToBase64Url(bytes){
  let out = '';
  for(let i=0;i<bytes.length;i+=3){
    const b0=bytes[i], b1=bytes[i+1], b2=bytes[i+2];
    const has1 = i+1<bytes.length, has2 = i+2<bytes.length;
    out += B64URL_CHARS[b0>>2];
    out += B64URL_CHARS[((b0&3)<<4) | (has1? b1>>4 : 0)];
    if(has1) out += B64URL_CHARS[((b1&15)<<2) | (has2? b2>>6 : 0)];
    if(has2) out += B64URL_CHARS[b2&63];
  }
  return out;
}
function base64UrlToBytes(s){
  const rev = {};
  for(let i=0;i<B64URL_CHARS.length;i++) rev[B64URL_CHARS[i]] = i;
  const bytes = [];
  let buffer = 0, bits = 0;
  for(const ch of s){
    if(!(ch in rev)) throw new Error('invalid base64url character');
    buffer = (buffer << 6) | rev[ch];
    bits += 6;
    if(bits >= 8){
      bits -= 8;
      bytes.push((buffer >> bits) & 0xff);
    }
  }
  return bytes;
}

function compressString(str){
  return bytesToBase64Url(varintEncode(lzwCompress(str)));
}
function decompressString(code){
  return lzwDecompress(varintDecode(base64UrlToBytes(code)));
}
/* ===== SHARE-CODE-CODEC:END ===== */
```

- [ ] **Step 2: Verify syntax**

Run:
```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 3: Write and run the compressor round-trip test**

```bash
cat > /tmp/fcc_codec_test.js <<'EOF'
const fs = require('fs');
const assert = require('assert');
const html = fs.readFileSync(process.argv[2], 'utf8');
const m = html.match(/\/\* ===== SHARE-CODE-CODEC:START ===== \*\/([\s\S]*?)\/\* ===== SHARE-CODE-CODEC:END ===== \*\//);
if(!m) throw new Error('codec section markers not found in index.html');
eval(m[1]);

// compressString/decompressString round trip
assert.strictEqual(decompressString(compressString('')), '', 'empty string round trip');
assert.strictEqual(decompressString(compressString('a')), 'a', 'single char round trip');
assert.strictEqual(decompressString(compressString('เนยเค็ม 200 กรัม เนยเค็ม 200 กรัม')), 'เนยเค็ม 200 กรัม เนยเค็ม 200 กรัม', 'Thai text with repetition round trip');
assert.strictEqual(decompressString(compressString('🐰🐰🐰')), '🐰🐰🐰', 'astral/emoji characters round trip');
const longRepeated = 'บราวนี่เนยสด,'.repeat(50);
assert.strictEqual(decompressString(compressString(longRepeated)), longRepeated, 'long repeated text round trip');

// compression should meaningfully shrink repetitive text
assert.ok(compressString(longRepeated).length < longRepeated.length, 'repeated text compresses smaller than input');

// varint round trip including multi-byte codes
assert.deepStrictEqual(varintDecode(varintEncode([0, 1, 127, 128, 300, 65536, 1000000])), [0, 1, 127, 128, 300, 65536, 1000000], 'varint round trip incl. large codes');

// base64url round trip incl. all byte values
const allBytes = Array.from({length:256}, (_, i) => i);
assert.deepStrictEqual(base64UrlToBytes(bytesToBase64Url(allBytes)), allBytes, 'base64url round trip over all byte values');
assert.ok(!/[+/=]/.test(bytesToBase64Url(allBytes)), 'base64url output has no +, /, or = characters');

console.log('ALL CODEC TESTS PASSED');
EOF
node /tmp/fcc_codec_test.js index.html
```
Expected: `ALL CODEC TESTS PASSED` with exit code 0. If any assertion fails, fix the implementation in `index.html` (not the test) and re-run.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add LZW compressor primitives for recipe share codes

Pure, dependency-free compression (LZW dictionary substitution +
varint byte packing + URL-safe base64) that a later task wires up to
the actual share/import flow. Verified with a round-trip test covering
empty strings, Thai text, emoji/astral characters, and all 256 byte
values through the base64url alphabet.
EOF
)"
```

---

## Task 2: Share code envelope (encode/decode with validation)

**Files:**
- Modify: `index.html` — add immediately after the `/* ===== SHARE-CODE-CODEC:END ===== */` line added in Task 1.

**Interfaces:**
- Consumes: `compressString(str)`, `decompressString(code)` from Task 1.
- Produces: `SHARE_CODE_PREFIX` (string `'FCC1:'`), `SHARE_CODE_VERSION` (number `1`), `encodeShareCode(payload) -> string`, `decodeShareCode(code) -> {ok:true, payload} | {ok:false, error:string}`.

- [ ] **Step 1: Add the envelope functions**

Insert this block right after `/* ===== SHARE-CODE-CODEC:END ===== */`:

```js
/* ===== SHARE-CODE-ENVELOPE:START ===== */
const SHARE_CODE_PREFIX = 'FCC1:';
const SHARE_CODE_VERSION = 1;

function encodeShareCode(payload){
  const json = JSON.stringify(payload);
  return SHARE_CODE_PREFIX + compressString(json);
}

function decodeShareCode(code){
  const trimmed = String(code||'').trim();
  if(!trimmed.startsWith(SHARE_CODE_PREFIX)){
    return {ok:false, error:'ไม่ใช่โค้ดสูตรอาหารที่ถูกต้อง'};
  }
  const body = trimmed.slice(SHARE_CODE_PREFIX.length);
  let json;
  try{
    json = decompressString(body);
  }catch(e){
    return {ok:false, error:'โค้ดไม่ถูกต้องหรือเสียหาย'};
  }
  let payload;
  try{
    payload = JSON.parse(json);
  }catch(e){
    return {ok:false, error:'โค้ดไม่ถูกต้องหรือเสียหาย'};
  }
  if(!payload || typeof payload!=='object' || payload.type!=='recipe-bundle' || !Array.isArray(payload.recipes) || !payload.recipes.length){
    return {ok:false, error:'โค้ดไม่ถูกต้องหรือเสียหาย'};
  }
  if(typeof payload.v!=='number' || payload.v > SHARE_CODE_VERSION){
    return {ok:false, error:'โค้ดนี้สร้างจากแอปเวอร์ชันใหม่กว่า กรุณาอัปเดตแอป'};
  }
  return {ok:true, payload};
}
/* ===== SHARE-CODE-ENVELOPE:END ===== */
```

- [ ] **Step 2: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 3: Write and run the envelope test**

```bash
cat > /tmp/fcc_envelope_test.js <<'EOF'
const fs = require('fs');
const assert = require('assert');
const html = fs.readFileSync(process.argv[2], 'utf8');
const codecMatch = html.match(/\/\* ===== SHARE-CODE-CODEC:START ===== \*\/([\s\S]*?)\/\* ===== SHARE-CODE-CODEC:END ===== \*\//);
const envMatch = html.match(/\/\* ===== SHARE-CODE-ENVELOPE:START ===== \*\/([\s\S]*?)\/\* ===== SHARE-CODE-ENVELOPE:END ===== \*\//);
if(!codecMatch) throw new Error('codec section markers not found');
if(!envMatch) throw new Error('envelope section markers not found');
eval(codecMatch[1]);
eval(envMatch[1]);

// happy path round trip
const payload = {v:1, type:'recipe-bundle', packaging:[], recipes:[{name:'บราวนี่', ingredients:[]}]};
const code = encodeShareCode(payload);
assert.ok(code.startsWith('FCC1:'), 'encoded code has the FCC1: prefix');
const decoded = decodeShareCode(code);
assert.strictEqual(decoded.ok, true, 'valid code decodes ok');
assert.deepStrictEqual(decoded.payload, payload, 'decoded payload matches original');

// wrong prefix
const noPrefix = decodeShareCode('not a real code');
assert.strictEqual(noPrefix.ok, false);
assert.strictEqual(noPrefix.error, 'ไม่ใช่โค้ดสูตรอาหารที่ถูกต้อง');

// corrupted body (valid prefix, garbage base64url payload)
const corrupted = decodeShareCode('FCC1:@@@not-valid-base64@@@');
assert.strictEqual(corrupted.ok, false);
assert.strictEqual(corrupted.error, 'โค้ดไม่ถูกต้องหรือเสียหาย');

// truncated valid code (chops the encoded body mid-stream)
const truncated = decodeShareCode(code.slice(0, code.length - 5));
assert.strictEqual(truncated.ok, false);

// valid envelope, wrong shape payload
const wrongShape = decodeShareCode(encodeShareCode({v:1, type:'something-else'}));
assert.strictEqual(wrongShape.ok, false);
assert.strictEqual(wrongShape.error, 'โค้ดไม่ถูกต้องหรือเสียหาย');

// empty recipes array
const emptyRecipes = decodeShareCode(encodeShareCode({v:1, type:'recipe-bundle', packaging:[], recipes:[]}));
assert.strictEqual(emptyRecipes.ok, false);

// version too new
const tooNew = decodeShareCode(encodeShareCode({v:99, type:'recipe-bundle', packaging:[], recipes:[{name:'x',ingredients:[]}]}));
assert.strictEqual(tooNew.ok, false);
assert.strictEqual(tooNew.error, 'โค้ดนี้สร้างจากแอปเวอร์ชันใหม่กว่า กรุณาอัปเดตแอป');

// whitespace-padded code still decodes (paste often carries stray newlines)
const padded = decodeShareCode('  \n' + code + '\n  ');
assert.strictEqual(padded.ok, true);

console.log('ALL ENVELOPE TESTS PASSED');
EOF
node /tmp/fcc_envelope_test.js index.html
```
Expected: `ALL ENVELOPE TESTS PASSED` with exit code 0.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add FCC1 share code envelope with version/shape validation

encodeShareCode/decodeShareCode wrap the Task 1 codec with a prefixed,
versioned envelope so import can reject non-matching paste content,
corrupted codes, and future-version codes with a clear message instead
of decoding into garbage.
EOF
)"
```

---

## Task 3: Build share payload from selected recipes

**Files:**
- Modify: `index.html` — add after `decodeShareCode` (end of the envelope section from Task 2), before `calcRecipe` (currently line 343 area — insert after the envelope block, which will now sit just above it).

**Interfaces:**
- Consumes: `SHARE_CODE_VERSION` (Task 2); `state.recipes`/`state.pantry`/`state.packaging`, `findRecipe`, `findPackaging`, `ingName`, `ingUnitLabel`, `ingUnitCost`, `packagingUnitCost` (all existing).
- Produces: `buildSharePayload(recipeIds: string[]) -> object` matching the spec's payload shape.

- [ ] **Step 1: Add `buildSharePayload`**

Insert this immediately after `/* ===== SHARE-CODE-ENVELOPE:END ===== */`:

```js
/* ============================== SHARE CODE: BUILD PAYLOAD ============================== */
function buildSharePayload(recipeIds){
  const packagingMap = new Map(); // dedupe key -> {key, name, unitCost}
  let pkCounter = 0;

  function packagingKeyFor(r){
    if(r.packagingId){
      const pkg = findPackaging(r.packagingId);
      if(!pkg) return null;
      const dedupeKey = 'id:'+r.packagingId;
      if(!packagingMap.has(dedupeKey)){
        packagingMap.set(dedupeKey, {key:'pk'+(pkCounter++), name:pkg.name, unitCost:packagingUnitCost(pkg)});
      }
      return packagingMap.get(dedupeKey).key;
    }
    if(r.unlinkedPackaging){
      // dedupe by value too, so re-sharing an already-imported (still
      // unmapped) recipe alongside others using the same packaging doesn't
      // duplicate the packaging entry in the outgoing payload
      const dedupeKey = 'name:'+r.unlinkedPackaging.name+'|cost:'+r.unlinkedPackaging.unitCost;
      if(!packagingMap.has(dedupeKey)){
        packagingMap.set(dedupeKey, {key:'pk'+(pkCounter++), name:r.unlinkedPackaging.name, unitCost:r.unlinkedPackaging.unitCost});
      }
      return packagingMap.get(dedupeKey).key;
    }
    return null;
  }

  const recipes = recipeIds.map(id=>{
    const r = findRecipe(id);
    return {
      name: r.name,
      category: r.category,
      yieldQty: Number(r.yieldQty)||0,
      yieldUnit: r.yieldUnit,
      laborRate: Number(r.laborRate)||0,
      laborHours: Number(r.laborHours)||0,
      gasPct: Number(r.gasPct)||0,
      deliveryCost: Number(r.deliveryCost)||0,
      overheadPct: Number(r.overheadPct)||0,
      targetPct: Number(r.targetPct)||0,
      packagingKey: packagingKeyFor(r),
      ingredients: r.ingredients.map(ing=>({
        name: ingName(ing),
        qty: Number(ing.qty)||0,
        unit: ingUnitLabel(ing),
        unitCost: ingUnitCost(ing),
      })),
    };
  });

  return {
    v: SHARE_CODE_VERSION,
    type: 'recipe-bundle',
    packaging: Array.from(packagingMap.values()),
    recipes,
  };
}
```

- [ ] **Step 2: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 3: Browser-verify `buildSharePayload`**

Open `index.html` in the Browser tool, then run each assertion below via the JS execution tool and confirm the printed result matches what's expected.

Build fixtures (pantry item, packaging item, one recipe using both, one "no packaging" recipe):
```js
const pantryItem = {id:uid('p'), name:'เนยเค็ม', category:PANTRY_CATS[0], family:'mass', buyPrice:90, buyQty:1000, buyUnit:'กรัม'};
state.pantry.push(pantryItem);
const pkgItem = {id:uid('pk'), name:'แก้วกาแฟ', buyPrice:1000, buyQty:200, buyUnit:'ชิ้น', piecesPerBox:100};
state.packaging.push(pkgItem);

const r1 = {id:uid('r'), name:'บราวนี่', category:'ขนม', yieldQty:10, yieldUnit:'ชิ้น',
  ingredients:[{id:uid('i'), mode:'pantry', pantryId:pantryItem.id, qty:200}],
  packagingId:pkgItem.id, unlinkedPackaging:null,
  laborRate:60, laborHours:1, gasPct:5, deliveryCost:0, overheadPct:20, targetPct:40, sellPrice:99, note:''};
const r2 = {id:uid('r'), name:'ข้าวผัด', category:'อาหาร', yieldQty:1, yieldUnit:'จาน',
  ingredients:[], packagingId:'', unlinkedPackaging:null,
  laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:30, sellPrice:0, note:''};
state.recipes.push(r1, r2);
JSON.stringify({r1id:r1.id, r2id:r2.id});
```
Expected: prints the two generated ids (just confirms setup ran without error).

Now build and inspect the payload:
```js
const payload = buildSharePayload([r1.id, r2.id]);
JSON.stringify(payload, null, 2);
```
Expected: an object where —
- `payload.v === 1` and `payload.type === 'recipe-bundle'`.
- `payload.packaging` has exactly one entry: `{key:'pk0', name:'แก้วกาแฟ', unitCost:5}` (1000 บาท / (200 ชิ้น / 100 ชิ้นต่อกล่อง) = 5 บาท/ชิ้น — see `packagingUnitCost`).
- `payload.recipes[0].name === 'บราวนี่'`, `packagingKey === 'pk0'`, `ingredients[0]` is `{name:'เนยเค็ม', qty:200, unit:'กรัม', unitCost:0.09}` (90/1000), and `sellPrice`/`note` are **not present** on the recipe object.
- `payload.recipes[1].name === 'ข้าวผัด'`, `packagingKey === null` (the "no packaging" recipe), `ingredients` is `[]`.

Confirm the excluded-field claim explicitly:
```js
JSON.stringify({hasSellPrice: 'sellPrice' in buildSharePayload([r1.id]).recipes[0], hasNote: 'note' in buildSharePayload([r1.id]).recipes[0]});
```
Expected: `{"hasSellPrice":false,"hasNote":false}`.

Confirm packaging dedup when two recipes share the same packaging item:
```js
const r3 = JSON.parse(JSON.stringify(r1));
r3.id = uid('r'); r3.name = 'บราวนี่ 2';
state.recipes.push(r3);
buildSharePayload([r1.id, r3.id]).packaging.length;
```
Expected: `1` (not 2 — same packaging item, deduped).

Clean up the fixtures so later tasks/tests start from a clean slate:
```js
state.recipes = state.recipes.filter(r => ![r1.id, r2.id, r3.id].includes(r.id));
state.pantry = state.pantry.filter(p => p.id !== pantryItem.id);
state.packaging = state.packaging.filter(p => p.id !== pkgItem.id);
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add buildSharePayload to snapshot recipes for sharing

Ingredients and packaging are captured by value (name + resolved unit
cost), never by pantry/packaging ID, since those IDs are meaningless
on a friend's device. Packaging is deduped once per bundle so several
selected recipes sharing the same item don't repeat it. sellPrice and
note are intentionally excluded — verified via buildSharePayload
against hand-built fixtures, including the packaging-dedup case and
the "no packaging" (packagingKey: null) case.
EOF
)"
```

---

## Task 4: Apply an imported bundle

**Files:**
- Modify: `index.html` — add right after `buildSharePayload` from Task 3.

**Interfaces:**
- Consumes: `uid`, `RECIPE_CATS`, `state.recipes` (existing).
- Produces: `applyImportedBundle(payload) -> newRecipeObject[]`. Pushes the new recipes into `state.recipes` (prepended) as a side effect.

- [ ] **Step 1: Add `applyImportedBundle`**

```js
/* ============================== SHARE CODE: APPLY IMPORTED BUNDLE ============================== */
function applyImportedBundle(payload){
  const packagingByKey = {};
  (payload.packaging||[]).forEach(p=>{ packagingByKey[p.key] = p; });

  const newRecipes = payload.recipes.map(rp=>{
    const pkgSnapshot = rp.packagingKey ? packagingByKey[rp.packagingKey] : null;
    return {
      id: uid('r'),
      name: rp.name || 'สูตรนำเข้า',
      category: RECIPE_CATS.includes(rp.category) ? rp.category : RECIPE_CATS[0],
      yieldQty: Number(rp.yieldQty)||0,
      yieldUnit: rp.yieldUnit || 'ชิ้น',
      ingredients: (rp.ingredients||[]).map(ing=>({
        id: uid('i'),
        mode: 'unlinked',
        name: ing.name || '(ไม่มีชื่อ)',
        qty: Number(ing.qty)||0,
        unit: ing.unit || 'หน่วย',
        unitCost: Number(ing.unitCost)||0,
      })),
      packagingId: '',
      unlinkedPackaging: pkgSnapshot ? {name:pkgSnapshot.name, unitCost:Number(pkgSnapshot.unitCost)||0} : null,
      laborRate: Number(rp.laborRate)||0,
      laborHours: Number(rp.laborHours)||0,
      gasPct: Number(rp.gasPct)||0,
      deliveryCost: Number(rp.deliveryCost)||0,
      overheadPct: Number(rp.overheadPct)||0,
      targetPct: Number(rp.targetPct)||0,
      sellPrice: 0,
      note: '',
    };
  });

  state.recipes = [...newRecipes, ...state.recipes];
  return newRecipes;
}
```

- [ ] **Step 2: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 3: Browser-verify `applyImportedBundle`**

In the same running app (fresh `index.html` load is fine too), run:

```js
const beforeCount = state.recipes.length;
const payload = {
  v: 1, type: 'recipe-bundle',
  packaging: [{key:'pk0', name:'แก้วกาแฟ', unitCost:5}],
  recipes: [
    {name:'บราวนี่นำเข้า', category:'ขนม', yieldQty:10, yieldUnit:'ชิ้น',
     laborRate:60, laborHours:1, gasPct:5, deliveryCost:0, overheadPct:20, targetPct:40,
     packagingKey:'pk0', ingredients:[{name:'เนยเค็ม', qty:200, unit:'กรัม', unitCost:0.09}]},
    {name:'ข้าวผัดนำเข้า', category:'อาหาร', yieldQty:1, yieldUnit:'จาน',
     laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:30,
     packagingKey:null, ingredients:[]},
  ],
};
const imported = applyImportedBundle(payload);
JSON.stringify({
  countIncreasedByTwo: state.recipes.length === beforeCount + 2,
  firstIsNewest: state.recipes[0].id === imported[0].id,
  ing0Mode: imported[0].ingredients[0].mode,
  ing0UnitCost: imported[0].ingredients[0].unitCost,
  pkg0: imported[0].unlinkedPackaging,
  packagingId0: imported[0].packagingId,
  pkg1: imported[1].unlinkedPackaging,
  packagingId1: imported[1].packagingId,
  sellPrice0: imported[0].sellPrice,
  freshIdsDiffer: imported[0].id !== imported[1].id,
});
```
Expected:
```json
{
  "countIncreasedByTwo": true,
  "firstIsNewest": true,
  "ing0Mode": "unlinked",
  "ing0UnitCost": 0.09,
  "pkg0": {"name":"แก้วกาแฟ","unitCost":5},
  "packagingId0": "",
  "pkg1": null,
  "packagingId1": "",
  "sellPrice0": 0,
  "freshIdsDiffer": true
}
```
The `pkg1`/`packagingId1` result is the key check from the design spec: the second recipe's `packagingKey` was `null` (no packaging), and it must come back as `unlinkedPackaging:null, packagingId:''` — exactly the same shape as a locally-created "no packaging" recipe, not some pending/unlinked state.

Re-run `applyImportedBundle(payload)` a second time and confirm duplicates are allowed (import never dedupes):
```js
const before2 = state.recipes.length;
applyImportedBundle(payload);
state.recipes.length === before2 + 2;
```
Expected: `true`.

Clean up:
```js
state.recipes = state.recipes.filter(r => !r.name.includes('นำเข้า'));
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add applyImportedBundle to turn a decoded payload into new recipes

Ingredients come back as mode:'unlinked' (cost calculates immediately
off the snapshot); packaging comes back as either a linked-looking
unlinkedPackaging snapshot or, when the source recipe had no packaging
at all, the exact same packagingId:''/unlinkedPackaging:null shape a
locally-created "no packaging" recipe already has. Imported recipes
always get fresh ids and are prepended — re-importing the same code
never overwrites or dedupes.
EOF
)"
```

---

## Task 5: `calcRecipe` unlinked-packaging fallback + `recipeNeedsMapping`

**Files:**
- Modify: `index.html:346-347` (inside `calcRecipe`) and add a new function after `calcRecipe` (currently ends at line 376, right before `/* ============================== RENDER ROOT ============================== */`).

**Interfaces:**
- Produces: `recipeNeedsMapping(r) -> boolean`. `calcRecipe` behavior extended (same signature/return shape).

- [ ] **Step 1: Patch `calcRecipe`'s packaging cost line**

In `index.html`, find:
```js
  const pkg = r.packagingId ? findPackaging(r.packagingId) : null;
  const packUnitCost = pkg ? packagingUnitCost(pkg) : 0;
```
Replace with:
```js
  const pkg = r.packagingId ? findPackaging(r.packagingId) : null;
  const packUnitCost = pkg ? packagingUnitCost(pkg) : (r.unlinkedPackaging ? Number(r.unlinkedPackaging.unitCost)||0 : 0);
```

- [ ] **Step 2: Add `recipeNeedsMapping`**

Immediately after the closing `}` of `calcRecipe` (the line `    revenue,profit,margin,profitBasic,marginBasic,compare,yieldQty};` followed by `}`), insert:

```js

function recipeNeedsMapping(r){
  return r.ingredients.some(i => i.mode !== 'pantry') || !!r.unlinkedPackaging;
}
```

- [ ] **Step 3: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 4: Browser-verify**

```js
const rWithUnlinkedPkg = {id:uid('r'), name:'t1', yieldQty:10, ingredients:[{mode:'pantry', pantryId:'nope', qty:0}],
  packagingId:'', unlinkedPackaging:{name:'แก้ว', unitCost:5},
  laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, sellPrice:0};
const c1 = calcRecipe(rWithUnlinkedPkg);

const rNoPackaging = {id:uid('r'), name:'t2', yieldQty:10, ingredients:[],
  packagingId:'', unlinkedPackaging:null,
  laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, sellPrice:0};

const rWithUnlinkedIngredient = {id:uid('r'), name:'t3', yieldQty:10,
  ingredients:[{mode:'unlinked', name:'x', qty:1, unit:'กรัม', unitCost:2}],
  packagingId:'', unlinkedPackaging:null,
  laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, sellPrice:0};

JSON.stringify({
  packCostFromSnapshot: c1.pack,
  needsMapping_unlinkedPkg: recipeNeedsMapping(rWithUnlinkedPkg),
  needsMapping_noPackaging: recipeNeedsMapping(rNoPackaging),
  needsMapping_unlinkedIngredient: recipeNeedsMapping(rWithUnlinkedIngredient),
});
```
Expected: `{"packCostFromSnapshot":50,"needsMapping_unlinkedPkg":true,"needsMapping_noPackaging":false,"needsMapping_unlinkedIngredient":true}`.

`packCostFromSnapshot` is `5 บาท/ชิ้น × 10 yieldQty = 50` — proving `calcRecipe` uses the unlinked snapshot when there's no `packagingId`. `needsMapping_noPackaging: false` is the critical check: a recipe with genuinely no packaging must **not** be flagged as needing mapping.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Support unlinked packaging in cost calc; add recipeNeedsMapping

calcRecipe falls back to the unlinkedPackaging snapshot's cost when a
recipe has no packagingId yet but does have a pending snapshot from
import. recipeNeedsMapping(r) is the single source of truth for
whether a recipe still needs attention — true for any unlinked
ingredient or a pending packaging snapshot, false for a recipe that
genuinely uses no packaging at all.
EOF
)"
```

---

## Task 6: Recipe list — needs-mapping badge, share button, share modal

**Files:**
- Modify: `index.html`
  - CSS: add modal styles near the existing `.edit-panel` rule (around line 160).
  - `recipeGridInner` (lines 460-486): add badge + "แชร์" button.
  - `recipesListHTML` (lines 444-454): render the share modal when open.
  - Add new functions: `shareRecipe`, `closeShareModal`, `copyShareCode`, `fallbackCopy`, `shareModalHTML`.
  - `ui` object (around line 231-233 in the current file — confirm exact location with `grep -n "let ui = " index.html`): add `showShareModal:false, shareCode:''`.

**Interfaces:**
- Consumes: `buildSharePayload` (Task 3), `encodeShareCode` (Task 2), `recipeNeedsMapping` (Task 5).
- Produces: `shareRecipe(id)`, `closeShareModal()`, `copyShareCode()`, `shareModalHTML()` — all consumed by Task 7 (selection mode reuses the modal).

- [ ] **Step 1: Add modal CSS**

Find the line:
```css
.edit-panel{background:var(--color-accent-100);border-radius:20px;padding:16px 18px;margin-bottom:14px;display:flex;flex-direction:column;gap:12px}
```
Add immediately after it:
```css
.modal-overlay{position:fixed;inset:0;background:rgba(46,43,37,.55);display:flex;align-items:center;justify-content:center;padding:20px;z-index:50}
.modal-panel{background:var(--color-surface-2);border-radius:var(--radius-lg);padding:20px 22px;max-width:480px;width:100%;box-shadow:var(--shadow-lg);max-height:85vh;overflow-y:auto;display:flex;flex-direction:column;gap:10px}
```

- [ ] **Step 2: Add `showShareModal`/`shareCode` to `ui`**

Find:
```js
let ui = { tab:'recipes', recipeId:null, search:'', pantrySearch:'', pantryEditing:null, pantryAdding:false,
  packagingSearch:'', packagingEditing:null, packagingAdding:false,
  quick:{ mat:100, pack:10, labor:30, gas:10, delivery:0, overhead:15, yieldQty:10, target:35, sell:0 } };
```
Replace with:
```js
let ui = { tab:'recipes', recipeId:null, search:'', pantrySearch:'', pantryEditing:null, pantryAdding:false,
  packagingSearch:'', packagingEditing:null, packagingAdding:false,
  showShareModal:false, shareCode:'',
  quick:{ mat:100, pack:10, labor:30, gas:10, delivery:0, overhead:15, yieldQty:10, target:35, sell:0 } };
```

- [ ] **Step 3: Add the share functions**

Add this block right after `function deleteRecipe(id){...}` (which currently ends the "RECIPES: LIST" section, just before `/* ============================== RECIPES: EDITOR ============================== */`):

```js
function shareRecipe(id){
  ui.shareCode = encodeShareCode(buildSharePayload([id]));
  ui.showShareModal = true;
  render();
}
function closeShareModal(){
  ui.showShareModal = false;
  ui.shareCode = '';
  render();
}
function shareModalHTML(){
  return `
  <div class="modal-overlay" onclick="closeShareModal()">
    <div class="modal-panel" onclick="event.stopPropagation()">
      <h3 style="font-size:16px">โค้ดสูตรอาหาร</h3>
      <div class="muted" style="font-size:12.5px">คัดลอกโค้ดนี้ส่งให้เพื่อน แล้วให้เพื่อนวางในช่อง "นำเข้าโค้ด"</div>
      <textarea class="input" id="shareCodeText" readonly rows="6" style="font-family:monospace;font-size:12px;resize:vertical">${esc(ui.shareCode)}</textarea>
      <div style="display:flex;gap:8px;justify-content:flex-end">
        <button class="btn btn-secondary btn-sm" onclick="closeShareModal()">ปิด</button>
        <button class="btn btn-primary btn-sm" onclick="copyShareCode()">คัดลอก</button>
      </div>
      <div class="muted" id="shareCopyStatus" style="font-size:12px;min-height:16px"></div>
    </div>
  </div>`;
}
function fallbackCopy(ta, showStatus){
  ta.focus();
  ta.select();
  try{
    const ok = document.execCommand('copy');
    showStatus(ok ? 'คัดลอกแล้ว' : 'แตะเพื่อเลือกแล้วคัดลอกเอง (Ctrl/Cmd+C)');
  }catch(e){
    showStatus('แตะเพื่อเลือกแล้วคัดลอกเอง (Ctrl/Cmd+C)');
  }
}
function copyShareCode(){
  const ta = document.getElementById('shareCodeText');
  const status = document.getElementById('shareCopyStatus');
  const showStatus = (msg)=>{ if(status) status.textContent = msg; };
  if(navigator.clipboard && navigator.clipboard.writeText){
    navigator.clipboard.writeText(ta.value).then(()=>showStatus('คัดลอกแล้ว')).catch(()=>fallbackCopy(ta, showStatus));
  } else {
    fallbackCopy(ta, showStatus);
  }
}
```

- [ ] **Step 4: Render the modal from `recipesListHTML`**

Find:
```js
function recipesListHTML(){
  const q = ui.search.trim().toLowerCase();
  const list = state.recipes.filter(r=> !q || r.name.toLowerCase().includes(q));
  return `
  <div class="toolbar">
    <input class="input" placeholder="ค้นหาเมนู เช่น บราวนี่" value="${esc(ui.search)}" oninput="ui.search=this.value; renderRecipeGrid()" />
    <span class="tag tag-neutral">ทั้งหมด ${state.recipes.length} สูตร</span>
  </div>
  <div class="recipe-grid" id="recipeGrid">${recipeGridInner(list)}</div>
  `;
}
```
Replace with:
```js
function recipesListHTML(){
  const q = ui.search.trim().toLowerCase();
  const list = state.recipes.filter(r=> !q || r.name.toLowerCase().includes(q));
  return `
  <div class="toolbar">
    <input class="input" placeholder="ค้นหาเมนู เช่น บราวนี่" value="${esc(ui.search)}" oninput="ui.search=this.value; renderRecipeGrid()" />
    <span class="tag tag-neutral">ทั้งหมด ${state.recipes.length} สูตร</span>
  </div>
  <div class="recipe-grid" id="recipeGrid">${recipeGridInner(list)}</div>
  ${ui.showShareModal ? shareModalHTML() : ''}
  `;
}
```
(This will be extended again in Task 7 and Task 8 for selection-mode toolbar and the import modal — keep those separate future edits in mind, don't be surprised the function gets touched more than once across tasks.)

- [ ] **Step 5: Add badge and "แชร์" button to `recipeGridInner`**

Find:
```js
    return `
    <div class="recipe-card" onclick="openRecipe('${r.id}')">
      <div style="display:flex;justify-content:space-between;align-items:flex-start;gap:8px">
        <div><span class="tag tag-outline">${esc(r.category)} · ${r.yieldQty} ${esc(r.yieldUnit)}/สูตร</span>
          <div class="card-title" style="font-family:var(--font-display);font-size:18px;margin-top:6px">${esc(r.name)}</div>
        </div>
        <span class="tag ${marginTag}">กำไร ${c.margin.toFixed(0)}%</span>
      </div>
      <div class="stats">
        <div class="stat"><div class="l">ต้นทุน/หน่วย</div><div class="v num">${num2(c.perPieceFull)}</div></div>
        <div class="stat"><div class="l">ราคาขาย</div><div class="v num">${num2(r.sellPrice)}</div></div>
        <div class="stat" style="background:var(--color-accent-2-100)"><div class="l" style="color:var(--color-accent-2-800)">กำไร/สูตร</div><div class="v num" style="color:var(--color-accent-2-800)">${num2(c.profit)}</div></div>
      </div>
      <div style="display:flex;gap:6px;margin-top:2px" onclick="event.stopPropagation()">
        <button class="btn btn-secondary btn-sm" onclick="duplicateRecipe('${r.id}')">ทำสำเนา</button>
        <button class="btn btn-danger btn-sm" onclick="deleteRecipe('${r.id}')" style="margin-left:auto">ลบ</button>
      </div>
    </div>`;
```
Replace with:
```js
    return `
    <div class="recipe-card" onclick="openRecipe('${r.id}')">
      <div style="display:flex;justify-content:space-between;align-items:flex-start;gap:8px">
        <div><span class="tag tag-outline">${esc(r.category)} · ${r.yieldQty} ${esc(r.yieldUnit)}/สูตร</span>
          ${recipeNeedsMapping(r) ? `<span class="tag tag-danger" style="margin-left:6px">ต้องเชื่อมวัตถุดิบ</span>` : ''}
          <div class="card-title" style="font-family:var(--font-display);font-size:18px;margin-top:6px">${esc(r.name)}</div>
        </div>
        <span class="tag ${marginTag}">กำไร ${c.margin.toFixed(0)}%</span>
      </div>
      <div class="stats">
        <div class="stat"><div class="l">ต้นทุน/หน่วย</div><div class="v num">${num2(c.perPieceFull)}</div></div>
        <div class="stat"><div class="l">ราคาขาย</div><div class="v num">${num2(r.sellPrice)}</div></div>
        <div class="stat" style="background:var(--color-accent-2-100)"><div class="l" style="color:var(--color-accent-2-800)">กำไร/สูตร</div><div class="v num" style="color:var(--color-accent-2-800)">${num2(c.profit)}</div></div>
      </div>
      <div style="display:flex;gap:6px;margin-top:2px" onclick="event.stopPropagation()">
        <button class="btn btn-secondary btn-sm" onclick="shareRecipe('${r.id}')">แชร์</button>
        <button class="btn btn-secondary btn-sm" onclick="duplicateRecipe('${r.id}')">ทำสำเนา</button>
        <button class="btn btn-danger btn-sm" onclick="deleteRecipe('${r.id}')" style="margin-left:auto">ลบ</button>
      </div>
    </div>`;
```

- [ ] **Step 6: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 7: Browser-verify**

Navigate to `index.html` in the Browser tool. Create one recipe and one with an unlinked ingredient (to check the badge), render, then inspect:

```js
newRecipe(); // creates and opens a normal recipe — has no ingredients, so no badge
closeEditor();
const rid = state.recipes[0].id;
const unlinked = {id:uid('r'), name:'นำเข้าทดสอบ', category:'ขนม', yieldQty:1, yieldUnit:'ชิ้น',
  ingredients:[{id:uid('i'), mode:'unlinked', name:'x', qty:1, unit:'กรัม', unitCost:1}],
  packagingId:'', unlinkedPackaging:null, laborRate:0, laborHours:0, gasPct:0, deliveryCost:0,
  overheadPct:0, targetPct:0, sellPrice:0, note:''};
state.recipes.unshift(unlinked);
render();
JSON.stringify({
  badgeCount: document.querySelectorAll('.tag-danger').length >= 1,
  firstCardHasBadge: document.querySelector('.recipe-card')?.textContent.includes('ต้องเชื่อมวัตถุดิบ'),
});
```
Expected: `{"badgeCount":true,"firstCardHasBadge":true}` (the unlinked recipe is unshifted to the front, so the first card should carry the badge).

Now open the share modal for the plain recipe and confirm it decodes back correctly:
```js
shareRecipe(rid);
const codeText = document.getElementById('shareCodeText').value;
const decoded = decodeShareCode(codeText);
JSON.stringify({startsWithPrefix: codeText.startsWith('FCC1:'), decodedOk: decoded.ok, recipeName: decoded.payload.recipes[0].name});
```
Expected: `{"startsWithPrefix":true,"decodedOk":true,"recipeName":"สูตรใหม่"}`.

Close it and confirm the modal is gone:
```js
closeShareModal();
document.querySelector('.modal-overlay') === null;
```
Expected: `true`.

Clean up:
```js
state.recipes = state.recipes.filter(r => r.id !== unlinked.id && r.id !== rid);
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add recipe list share button, needs-mapping badge, and share modal

Each recipe card gets a "แชร์" button that encodes just that recipe
and opens a modal with a read-only, copyable code. Cards with any
unlinked ingredient or pending packaging snapshot get a red
"ต้องเชื่อมวัตถุดิบ" badge, driven by recipeNeedsMapping. Copy tries
the Clipboard API first and falls back to execCommand('copy') since
the app is frequently opened via file://, where the Clipboard API is
often unavailable.
EOF
)"
```

---

## Task 7: Recipe list — selection mode (hold-tap, bundle share)

**Files:**
- Modify: `index.html`
  - CSS: add selection styles near `.recipe-card` (around line 108-110).
  - `ui` object: add `selectionMode:false, selectedIds:new Set()`.
  - `recipesListHTML`: swap toolbar to a selection bar when `ui.selectionMode`.
  - `recipeGridInner`: render checkbox overlay, route card clicks/long-press through selection-aware handlers, hide per-card action buttons and the "new recipe" card while selecting.
  - Add new functions: `startLongPress`, `cancelLongPress`, `enterSelectionMode`, `exitSelectionMode`, `toggleRecipeSelection`, `onRecipeCardClick`, `shareSelectedRecipes`, `selectionToolbarHTML`.

**Interfaces:**
- Consumes: `shareModalHTML`/`closeShareModal` (Task 6), `buildSharePayload` (Task 3), `encodeShareCode` (Task 2), `recipeNeedsMapping` (Task 5).

- [ ] **Step 1: Add selection CSS**

Find the `.recipe-card{...}` rule (around line 108) and its `:hover` rule (line 110). Add immediately after the `:hover` rule:
```css
.recipe-card{position:relative}
.recipe-card.selected{border-color:var(--color-accent);background:var(--color-accent-100)}
.select-check{position:absolute;top:12px;right:12px;width:22px;height:22px;border-radius:999px;
  border:2px solid var(--color-accent);display:flex;align-items:center;justify-content:center;
  font-size:13px;color:var(--color-accent);background:var(--color-surface-2)}
.select-check.checked{background:var(--color-accent);color:#fff}
```
(`.recipe-card{position:relative}` is safe to add even though the base rule may already imply static positioning — this later declaration wins and gives `.select-check` a positioning context either way.)

- [ ] **Step 2: Extend `ui`**

Find:
```js
let ui = { tab:'recipes', recipeId:null, search:'', pantrySearch:'', pantryEditing:null, pantryAdding:false,
  packagingSearch:'', packagingEditing:null, packagingAdding:false,
  showShareModal:false, shareCode:'',
  quick:{ mat:100, pack:10, labor:30, gas:10, delivery:0, overhead:15, yieldQty:10, target:35, sell:0 } };
```
Replace with:
```js
let ui = { tab:'recipes', recipeId:null, search:'', pantrySearch:'', pantryEditing:null, pantryAdding:false,
  packagingSearch:'', packagingEditing:null, packagingAdding:false,
  showShareModal:false, shareCode:'', selectionMode:false, selectedIds:new Set(),
  quick:{ mat:100, pack:10, labor:30, gas:10, delivery:0, overhead:15, yieldQty:10, target:35, sell:0 } };
```

- [ ] **Step 3: Add selection-mode functions**

Add this block right after `copyShareCode` (end of Task 6's additions):

```js
let longPressTimer = null;
function startLongPress(id){
  clearTimeout(longPressTimer);
  longPressTimer = setTimeout(()=>enterSelectionMode(id), 500);
}
function cancelLongPress(){ clearTimeout(longPressTimer); }
function enterSelectionMode(id){
  ui.selectionMode = true;
  ui.selectedIds = new Set([id]);
  render();
}
function exitSelectionMode(){
  ui.selectionMode = false;
  ui.selectedIds = new Set();
  render();
}
function toggleRecipeSelection(id){
  if(ui.selectedIds.has(id)) ui.selectedIds.delete(id); else ui.selectedIds.add(id);
  if(ui.selectedIds.size===0){ exitSelectionMode(); return; }
  render();
}
function onRecipeCardClick(id){
  if(ui.selectionMode){ toggleRecipeSelection(id); } else { openRecipe(id); }
}
function shareSelectedRecipes(){
  if(ui.selectedIds.size===0) return;
  ui.shareCode = encodeShareCode(buildSharePayload(Array.from(ui.selectedIds)));
  ui.showShareModal = true;
  ui.selectionMode = false;
  ui.selectedIds = new Set();
  render();
}
function selectionToolbarHTML(){
  return `
    <span class="tag tag-accent">เลือกแล้ว ${ui.selectedIds.size} สูตร</span>
    <div style="display:flex;gap:8px;margin-left:auto">
      <button class="btn btn-secondary btn-sm" onclick="exitSelectionMode()">ยกเลิก</button>
      <button class="btn btn-primary btn-sm" ${ui.selectedIds.size===0?'disabled':''} onclick="shareSelectedRecipes()">คัดลอกโค้ด</button>
    </div>
  `;
}
```

- [ ] **Step 4: Swap the toolbar in `recipesListHTML`**

Find (as left by Task 6):
```js
  <div class="toolbar">
    <input class="input" placeholder="ค้นหาเมนู เช่น บราวนี่" value="${esc(ui.search)}" oninput="ui.search=this.value; renderRecipeGrid()" />
    <span class="tag tag-neutral">ทั้งหมด ${state.recipes.length} สูตร</span>
  </div>
```
Replace with:
```js
  <div class="toolbar">
    ${ui.selectionMode ? selectionToolbarHTML() : `
    <input class="input" placeholder="ค้นหาเมนู เช่น บราวนี่" value="${esc(ui.search)}" oninput="ui.search=this.value; renderRecipeGrid()" />
    <span class="tag tag-neutral">ทั้งหมด ${state.recipes.length} สูตร</span>
    `}
  </div>
```

- [ ] **Step 5: Make `recipeGridInner` selection-aware**

Find the card template added/left by Task 6 (`<div class="recipe-card" onclick="openRecipe('${r.id}')">...`). Replace the whole `recipeGridInner` function with:

```js
function recipeGridInner(list){
  const cards = list.map(r=>{
    const c = calcRecipe(r);
    const marginTag = c.margin>=0 ? 'tag-accent-2' : 'tag-danger';
    const selected = ui.selectionMode && ui.selectedIds.has(r.id);
    return `
    <div class="recipe-card ${selected?'selected':''}" onclick="onRecipeCardClick('${r.id}')"
      onpointerdown="startLongPress('${r.id}')" onpointerup="cancelLongPress()" onpointerleave="cancelLongPress()" onpointercancel="cancelLongPress()">
      ${ui.selectionMode ? `<div class="select-check ${selected?'checked':''}">${selected?'✓':''}</div>` : ''}
      <div style="display:flex;justify-content:space-between;align-items:flex-start;gap:8px">
        <div><span class="tag tag-outline">${esc(r.category)} · ${r.yieldQty} ${esc(r.yieldUnit)}/สูตร</span>
          ${recipeNeedsMapping(r) ? `<span class="tag tag-danger" style="margin-left:6px">ต้องเชื่อมวัตถุดิบ</span>` : ''}
          <div class="card-title" style="font-family:var(--font-display);font-size:18px;margin-top:6px">${esc(r.name)}</div>
        </div>
        <span class="tag ${marginTag}">กำไร ${c.margin.toFixed(0)}%</span>
      </div>
      <div class="stats">
        <div class="stat"><div class="l">ต้นทุน/หน่วย</div><div class="v num">${num2(c.perPieceFull)}</div></div>
        <div class="stat"><div class="l">ราคาขาย</div><div class="v num">${num2(r.sellPrice)}</div></div>
        <div class="stat" style="background:var(--color-accent-2-100)"><div class="l" style="color:var(--color-accent-2-800)">กำไร/สูตร</div><div class="v num" style="color:var(--color-accent-2-800)">${num2(c.profit)}</div></div>
      </div>
      ${!ui.selectionMode ? `
      <div style="display:flex;gap:6px;margin-top:2px" onclick="event.stopPropagation()">
        <button class="btn btn-secondary btn-sm" onclick="shareRecipe('${r.id}')">แชร์</button>
        <button class="btn btn-secondary btn-sm" onclick="duplicateRecipe('${r.id}')">ทำสำเนา</button>
        <button class="btn btn-danger btn-sm" onclick="deleteRecipe('${r.id}')" style="margin-left:auto">ลบ</button>
      </div>` : ''}
    </div>`;
  }).join('');
  return cards + (ui.selectionMode ? '' : `<div class="newcard" onclick="newRecipe()">
    <svg width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><path d="M12 5v14M5 12h14"></path></svg>
    สร้างสูตรใหม่</div>`);
}
```

- [ ] **Step 6: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 7: Browser-verify**

```js
newRecipe(); closeEditor();
newRecipe(); closeEditor();
const [idA, idB] = [state.recipes[0].id, state.recipes[1].id];

enterSelectionMode(idA);
JSON.stringify({
  selectionModeOn: ui.selectionMode,
  selectedSize: ui.selectedIds.size,
  toolbarShowsCount: document.querySelector('.tag-accent')?.textContent.includes('เลือกแล้ว 1 สูตร'),
  noNewCard: document.querySelector('.newcard') === null,
});
```
Expected: `{"selectionModeOn":true,"selectedSize":1,"toolbarShowsCount":true,"noNewCard":true}`.

```js
toggleRecipeSelection(idB);
JSON.stringify({selectedSize: ui.selectedIds.size, hasBoth: ui.selectedIds.has(idA) && ui.selectedIds.has(idB)});
```
Expected: `{"selectedSize":2,"hasBoth":true}`.

```js
shareSelectedRecipes();
const decoded = decodeShareCode(document.getElementById('shareCodeText').value);
JSON.stringify({
  selectionModeOffAfterShare: ui.selectionMode,
  modalOpen: ui.showShareModal,
  bundledCount: decoded.payload.recipes.length,
});
```
Expected: `{"selectionModeOffAfterShare":false,"modalOpen":true,"bundledCount":2}`.

```js
closeShareModal();
toggleRecipeSelection(idA); // deselecting the only selected one (post-share selection was cleared, so re-select then deselect)
enterSelectionMode(idA);
toggleRecipeSelection(idA);
ui.selectionMode; // deselecting the last selected id must exit selection mode automatically
```
Expected: `false`.

Clean up:
```js
state.recipes = state.recipes.filter(r => r.id !== idA && r.id !== idB);
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add hold-tap multi-select mode for sharing several recipes at once

Long-pressing a recipe card (pointerdown + 500ms timer, cancelled on
pointerup/leave/cancel — works with touch and mouse) enters selection
mode: the toolbar swaps to a "เลือกแล้ว N สูตร" bar with
คัดลอกโค้ด/ยกเลิก, cards show a checkmark overlay, and tapping other
cards toggles them instead of opening the editor. คัดลอกโค้ด reuses
the Task 6 share modal with all selected recipes bundled into one
code. Deselecting the last selected card exits selection mode
automatically.
EOF
)"
```

---

## Task 8: Recipe list — import button and modal

**Files:**
- Modify: `index.html`
  - `ui` object: add `showImportModal:false, importError:''`.
  - `recipesListHTML`: add "นำเข้าโค้ด" button (non-selection-mode toolbar) and render the import modal.
  - Add new functions: `openImportModal`, `closeImportModal`, `submitImport`, `importModalHTML`.

**Interfaces:**
- Consumes: `decodeShareCode` (Task 2), `applyImportedBundle` (Task 4), `recipeNeedsMapping` (Task 5).

- [ ] **Step 1: Extend `ui`**

Find:
```js
let ui = { tab:'recipes', recipeId:null, search:'', pantrySearch:'', pantryEditing:null, pantryAdding:false,
  packagingSearch:'', packagingEditing:null, packagingAdding:false,
  showShareModal:false, shareCode:'', selectionMode:false, selectedIds:new Set(),
  quick:{ mat:100, pack:10, labor:30, gas:10, delivery:0, overhead:15, yieldQty:10, target:35, sell:0 } };
```
Replace with:
```js
let ui = { tab:'recipes', recipeId:null, search:'', pantrySearch:'', pantryEditing:null, pantryAdding:false,
  packagingSearch:'', packagingEditing:null, packagingAdding:false,
  showShareModal:false, shareCode:'', selectionMode:false, selectedIds:new Set(),
  showImportModal:false, importError:'',
  quick:{ mat:100, pack:10, labor:30, gas:10, delivery:0, overhead:15, yieldQty:10, target:35, sell:0 } };
```

- [ ] **Step 2: Add import functions**

Add this block right after `selectionToolbarHTML` (end of Task 7's additions):

```js
function openImportModal(){
  ui.showImportModal = true;
  ui.importError = '';
  render();
}
function closeImportModal(){
  ui.showImportModal = false;
  ui.importError = '';
  render();
}
function importModalHTML(){
  return `
  <div class="modal-overlay" onclick="closeImportModal()">
    <div class="modal-panel" onclick="event.stopPropagation()">
      <h3 style="font-size:16px">นำเข้าโค้ดสูตรอาหาร</h3>
      <div class="muted" style="font-size:12.5px">วางโค้ดที่เพื่อนส่งมาที่นี่</div>
      <textarea class="input" id="importCodeText" rows="6" style="font-family:monospace;font-size:12px;resize:vertical" placeholder="FCC1:..."></textarea>
      ${ui.importError ? `<div class="warn"><div>${esc(ui.importError)}</div></div>` : ''}
      <div style="display:flex;gap:8px;justify-content:flex-end">
        <button class="btn btn-secondary btn-sm" onclick="closeImportModal()">ยกเลิก</button>
        <button class="btn btn-primary btn-sm" onclick="submitImport()">นำเข้า</button>
      </div>
    </div>
  </div>`;
}
function submitImport(){
  const code = document.getElementById('importCodeText').value;
  const result = decodeShareCode(code);
  if(!result.ok){
    ui.importError = result.error;
    render();
    return;
  }
  const imported = applyImportedBundle(result.payload);
  ui.showImportModal = false;
  ui.importError = '';
  ui.recipeId = null;
  render();
  schedulePersist();
  const anyUnmapped = imported.some(recipeNeedsMapping);
  alert('นำเข้าสำเร็จ '+imported.length+' สูตร'+(anyUnmapped ? ' — บางสูตรต้องเชื่อมวัตถุดิบ/บรรจุภัณฑ์กับคลังของคุณ' : ''));
}
```

- [ ] **Step 3: Add the button and modal render to `recipesListHTML`**

Find (as left by Task 7):
```js
  <div class="toolbar">
    ${ui.selectionMode ? selectionToolbarHTML() : `
    <input class="input" placeholder="ค้นหาเมนู เช่น บราวนี่" value="${esc(ui.search)}" oninput="ui.search=this.value; renderRecipeGrid()" />
    <span class="tag tag-neutral">ทั้งหมด ${state.recipes.length} สูตร</span>
    `}
  </div>
  <div class="recipe-grid" id="recipeGrid">${recipeGridInner(list)}</div>
  ${ui.showShareModal ? shareModalHTML() : ''}
```
Replace with:
```js
  <div class="toolbar">
    ${ui.selectionMode ? selectionToolbarHTML() : `
    <input class="input" placeholder="ค้นหาเมนู เช่น บราวนี่" value="${esc(ui.search)}" oninput="ui.search=this.value; renderRecipeGrid()" />
    <span class="tag tag-neutral">ทั้งหมด ${state.recipes.length} สูตร</span>
    <button class="btn btn-secondary btn-sm" style="margin-left:auto" onclick="openImportModal()">นำเข้าโค้ด</button>
    `}
  </div>
  <div class="recipe-grid" id="recipeGrid">${recipeGridInner(list)}</div>
  ${ui.showShareModal ? shareModalHTML() : ''}
  ${ui.showImportModal ? importModalHTML() : ''}
```

- [ ] **Step 4: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 5: Browser-verify**

Test the error path first (no `alert()` involved, safe to automate):
```js
openImportModal();
document.getElementById('importCodeText').value = 'garbage not a code';
submitImport();
JSON.stringify({modalStillOpen: ui.showImportModal, errorShown: document.querySelector('.warn')?.textContent.includes('ไม่ใช่โค้ดสูตรอาหารที่ถูกต้อง')});
```
Expected: `{"modalStillOpen":true,"errorShown":true}`.

Now the success path. `submitImport()` calls `alert(...)` on success, which blocks the browser tool's JS execution — build and apply the payload directly via `applyImportedBundle` (already covered end-to-end in Task 4) and instead verify `decodeShareCode` + the modal's error-clearing behavior here, to keep this step non-blocking:
```js
closeImportModal();
const payload = {v:1, type:'recipe-bundle', packaging:[], recipes:[{name:'ทดสอบนำเข้า', category:'ขนม', yieldQty:1, yieldUnit:'ชิ้น', laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, packagingKey:null, ingredients:[]}]};
const code = encodeShareCode(payload);
const decoded = decodeShareCode(code);
JSON.stringify({decodedOk: decoded.ok, name: decoded.payload.recipes[0].name});
```
Expected: `{"decodedOk":true,"name":"ทดสอบนำเข้า"}`.

```js
openImportModal();
document.getElementById('importCodeText').value = 'garbage';
submitImport();
document.getElementById('importCodeText').value = code;
JSON.stringify({errorStillShownBeforeRetry: !!ui.importError});
```
Expected: `{"errorStillShownBeforeRetry":true}` (confirms the error from the first attempt is still on screen right up until a fresh submit — i.e. nothing clears it prematurely).

Finish the successful import manually via `applyImportedBundle` (equivalent to what `submitImport` does right before its `alert`, without triggering the blocking `alert`):
```js
const before = state.recipes.length;
applyImportedBundle(decoded.payload);
closeImportModal();
JSON.stringify({countIncreased: state.recipes.length === before + 1, modalClosed: !ui.showImportModal});
```
Expected: `{"countIncreased":true,"modalClosed":true}`.

Clean up:
```js
state.recipes = state.recipes.filter(r => r.name !== 'ทดสอบนำเข้า');
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add import button and paste-a-code modal to the recipe list

"นำเข้าโค้ด" opens a modal with a paste textarea; submitting runs the
code through decodeShareCode and, on success, applyImportedBundle,
closing the modal and summarizing how many recipes came in (and
whether any still need mapping). Invalid/corrupt codes show an inline
error in the modal instead of closing it.
EOF
)"
```

---

## Task 9: Recipe editor — unmapped banner + ingredient row mapping

**Files:**
- Modify: `index.html`
  - `recipeEditorHTML` (around lines 519-584): add a banner container right under the back button.
  - `ingRowHTML` (lines 590-603): branch rendering for `mode==='unlinked'`.
  - Add new functions: `unmappedBannerInner`, `refreshUnmappedBanner`, `mapIngredientToPantry`, `guessPantryFamilyFromUnit`, `addUnlinkedIngredientToPantry`.

**Interfaces:**
- Consumes: `recipeNeedsMapping` (Task 5), `FAMILY_UNITS`/`PANTRY_CATS` (existing), `refreshCalcPanel` (existing), `ingRowHTML`/`ingCost`/`ingUnitLabel` (existing).
- Produces: `mapIngredientToPantry(rid, idx, pantryId)`, `addUnlinkedIngredientToPantry(rid, idx)`, `refreshUnmappedBanner(rid)` — the last one also consumed by Task 10.

- [ ] **Step 1: Add the banner container to `recipeEditorHTML`**

Find:
```js
  return `
  <div class="editor-head">
    <button class="btn btn-ghost" onclick="closeEditor()">‹ กลับไปหน้าสูตรทั้งหมด</button>
  </div>
  <div class="editor-grid">
```
Replace with:
```js
  return `
  <div class="editor-head">
    <button class="btn btn-ghost" onclick="closeEditor()">‹ กลับไปหน้าสูตรทั้งหมด</button>
  </div>
  <div id="unmappedBanner_${id}">${unmappedBannerInner(r)}</div>
  <div class="editor-grid">
```

- [ ] **Step 2: Add banner + mapping functions**

Add this block right before `function ingTableInner(r){` (currently line 586):

```js
function unmappedBannerInner(r){
  if(!recipeNeedsMapping(r)) return '';
  const items = r.ingredients.filter(i=>i.mode!=='pantry').map(i=>i.name);
  if(r.unlinkedPackaging) items.push(r.unlinkedPackaging.name+' (บรรจุภัณฑ์)');
  return `<div class="warn" style="margin-bottom:16px">
    <svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="#b2622d" stroke-width="2.75" stroke-linecap="round"><circle cx="12" cy="12" r="9"></circle><path d="M12 8h.01M12 11v5"></path></svg>
    <div>สูตรนี้นำเข้ามาจากที่อื่น ยังไม่ได้เชื่อมกับคลังของคุณ: ${items.map(esc).join(', ')} — เลือกจากคลังหรือกด "+ เพิ่มเข้าคลัง" ที่แถวนั้นๆ</div>
  </div>`;
}
function refreshUnmappedBanner(rid){
  const r = findRecipe(rid);
  const el = document.getElementById('unmappedBanner_'+rid);
  if(r && el) el.innerHTML = unmappedBannerInner(r);
}
function mapIngredientToPantry(rid, idx, pantryId){
  const r = findRecipe(rid);
  const ing = r.ingredients[idx];
  ing.mode = 'pantry';
  ing.pantryId = pantryId;
  delete ing.name; delete ing.unit; delete ing.unitCost;
  const rowEl = document.getElementById('ingrow_'+rid+'_'+idx);
  if(rowEl) rowEl.outerHTML = ingRowHTML(r, idx);
  refreshCalcPanel(rid);
  refreshUnmappedBanner(rid);
  schedulePersist();
}
function guessPantryFamilyFromUnit(unit){
  if(unit==='กรัม'||unit==='กก.') return 'mass';
  if(unit==='มล.'||unit==='ลิตร') return 'volume';
  if(unit==='ฟอง') return 'egg';
  return 'piece';
}
function addUnlinkedIngredientToPantry(rid, idx){
  const r = findRecipe(rid);
  const ing = r.ingredients[idx];
  const family = guessPantryFamilyFromUnit(ing.unit);
  const buyUnit = FAMILY_UNITS[family].includes(ing.unit) ? ing.unit : FAMILY_UNITS[family][0];
  const newPantryItem = { id: uid('p'), name: ing.name, category: PANTRY_CATS[0], family, buyPrice: Number(ing.unitCost)||0, buyQty: 1, buyUnit };
  state.pantry.push(newPantryItem);
  mapIngredientToPantry(rid, idx, newPantryItem.id);
}
```

- [ ] **Step 3: Branch `ingRowHTML` for unlinked rows**

Find:
```js
function ingRowHTML(r, idx){
  const ing = r.ingredients[idx];
  const cost = ingCost(ing);
  return `<div class="ing-row" id="ingrow_${r.id}_${idx}">
    <select class="input" onchange="updIngField('${r.id}',${idx},'pantryId',this.value)">
      ${state.pantry.map(p=>`<option value="${p.id}" ${p.id===ing.pantryId?'selected':''}>${esc(p.name)}</option>`).join('')}
    </select>
    <input class="input" type="number" min="0" step="1" value="${ing.qty??''}" placeholder="ปริมาณ"
      oninput="updIngField('${r.id}',${idx},'qty',this.value)" />
    <div class="muted" style="font-size:12px">${esc(ingUnitLabel(ing))}</div>
    <div class="num" id="ingcost_${r.id}_${idx}" style="font-size:14px">${num2(cost)}</div>
    <button class="btn btn-ghost btn-sm" onclick="removeIngredient('${r.id}',${idx})" title="ลบ">✕</button>
  </div>`;
}
```
Replace with:
```js
function ingRowHTML(r, idx){
  const ing = r.ingredients[idx];
  const cost = ingCost(ing);
  const isUnlinked = ing.mode === 'unlinked';
  return `<div class="ing-row" id="ingrow_${r.id}_${idx}">
    <div>
      <select class="input" onchange="mapIngredientToPantry('${r.id}',${idx},this.value)">
        ${isUnlinked ? `<option value="" selected disabled>🟡 (นำเข้า) ${esc(ing.name)} — ${num2(ing.unitCost)} บาท/${esc(ing.unit)}</option>` : ''}
        ${state.pantry.map(p=>`<option value="${p.id}" ${!isUnlinked && p.id===ing.pantryId?'selected':''}>${esc(p.name)}</option>`).join('')}
      </select>
      ${isUnlinked ? `<button class="btn btn-ghost btn-sm" style="padding:2px 4px;font-size:11px" onclick="addUnlinkedIngredientToPantry('${r.id}',${idx})">+ เพิ่มเข้าคลัง</button>` : ''}
    </div>
    <input class="input" type="number" min="0" step="1" value="${ing.qty??''}" placeholder="ปริมาณ"
      oninput="updIngField('${r.id}',${idx},'qty',this.value)" />
    <div class="muted" style="font-size:12px">${esc(ingUnitLabel(ing))}</div>
    <div class="num" id="ingcost_${r.id}_${idx}" style="font-size:14px">${num2(cost)}</div>
    <button class="btn btn-ghost btn-sm" onclick="removeIngredient('${r.id}',${idx})" title="ลบ">✕</button>
  </div>`;
}
```

- [ ] **Step 4: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 5: Browser-verify**

Create a pantry item and a recipe with one unlinked ingredient, open it, and check the banner + row rendering:
```js
const pantryItem = {id:uid('p'), name:'เนยเค็ม', category:PANTRY_CATS[0], family:'mass', buyPrice:90, buyQty:1000, buyUnit:'กรัม'};
state.pantry.push(pantryItem);
const r = {id:uid('r'), name:'ทดสอบเชื่อม', category:'ขนม', yieldQty:1, yieldUnit:'ชิ้น',
  ingredients:[{id:uid('i'), mode:'unlinked', name:'เนยเค็ม', qty:200, unit:'กรัม', unitCost:0.09}],
  packagingId:'', unlinkedPackaging:null, laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, sellPrice:0, note:''};
state.recipes.unshift(r);
openRecipe(r.id);
JSON.stringify({
  bannerVisible: document.getElementById('unmappedBanner_'+r.id).textContent.includes('เนยเค็ม'),
  rowShowsPendingOption: document.querySelector('.ing-row select').selectedOptions[0].textContent.includes('🟡'),
  addToPantryLinkPresent: document.querySelector('.ing-row').textContent.includes('+ เพิ่มเข้าคลัง'),
});
```
Expected: `{"bannerVisible":true,"rowShowsPendingOption":true,"addToPantryLinkPresent":true}`.

Map it to the existing pantry item (simulating picking it from the dropdown):
```js
mapIngredientToPantry(r.id, 0, pantryItem.id);
JSON.stringify({
  ingredientMode: r.ingredients[0].mode,
  ingredientPantryId: r.ingredients[0].pantryId,
  bannerGone: document.getElementById('unmappedBanner_'+r.id).innerHTML.trim() === '',
});
```
Expected: `{"ingredientMode":"pantry","ingredientPantryId":"<pantryItem.id value>","bannerGone":true}` (the `pantryId` should equal the `pantryItem.id` string generated above).

Reset back to unlinked and test the "add to pantry" path instead:
```js
r.ingredients[0] = {id:uid('i'), mode:'unlinked', name:'น้ำตาลทราย', qty:50, unit:'กรัม', unitCost:0.05};
openRecipe(r.id); // re-render editor with the fresh unlinked ingredient
const pantryCountBefore = state.pantry.length;
addUnlinkedIngredientToPantry(r.id, 0);
JSON.stringify({
  pantryCountIncreased: state.pantry.length === pantryCountBefore + 1,
  newPantryItemFamily: state.pantry[state.pantry.length-1].family,
  newPantryItemBuyUnit: state.pantry[state.pantry.length-1].buyUnit,
  newPantryItemPrice: state.pantry[state.pantry.length-1].buyPrice,
  ingredientNowLinked: r.ingredients[0].mode === 'pantry' && r.ingredients[0].pantryId === state.pantry[state.pantry.length-1].id,
});
```
Expected: `{"pantryCountIncreased":true,"newPantryItemFamily":"mass","newPantryItemBuyUnit":"กรัม","newPantryItemPrice":0.05,"ingredientNowLinked":true}`.

Clean up:
```js
closeEditor();
state.recipes = state.recipes.filter(x => x.id !== r.id);
state.pantry = state.pantry.filter(p => p.name !== 'เนยเค็ม' && p.name !== 'น้ำตาลทราย');
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add unmapped banner and ingredient mapping to the recipe editor

Opening a recipe with any unlinked ingredient or pending packaging
snapshot shows a banner naming what still needs attention; it clears
itself automatically once everything's resolved. Unlinked ingredient
rows show a pinned, disabled "🟡 (นำเข้า) ..." option in the same
pantry dropdown used everywhere else — picking a real item from that
dropdown is the map action, no separate control needed — plus a
"+ เพิ่มเข้าคลัง" quick action that creates a new pantry item
pre-filled from the snapshot (best-guess unit family, buyQty:1) and
links the row to it immediately.
EOF
)"
```

---

## Task 10: Recipe editor — packaging field mapping

**Files:**
- Modify: `index.html`
  - `recipeEditorHTML`'s "ต้นทุนอื่น" packaging `<div class="field">` (lines 555-559): replace with a call to a new `packagingFieldHTML(r)`.
  - Add new functions: `packagingFieldHTML`, `mapPackagingToWarehouse`, `addUnlinkedPackagingToWarehouse`.

**Interfaces:**
- Consumes: `refreshUnmappedBanner` (Task 9), `refreshCalcPanel` (existing), `packagingUnitCost` (existing).

- [ ] **Step 1: Replace the packaging field markup**

Find:
```js
          <div class="field"><label>บรรจุภัณฑ์ที่ใช้</label>
            <select class="input" onchange="updRecipeField('${id}','packagingId',this.value)">
              <option value="">— ไม่ใช้บรรจุภัณฑ์ —</option>
              ${state.packaging.map(p=>`<option value="${p.id}" ${p.id===r.packagingId?'selected':''}>${esc(p.name)} (${packagingUnitCost(p).toFixed(2)} บาท/ชิ้น)</option>`).join('')}
            </select></div>
```
Replace with:
```js
          ${packagingFieldHTML(r)}
```

- [ ] **Step 2: Add packaging mapping functions**

Add this block right before `function ingTableInner(r){` (immediately alongside the Task 9 additions — place it right after `addUnlinkedIngredientToPantry`):

```js
function packagingFieldHTML(r){
  const isUnlinked = !!r.unlinkedPackaging;
  return `<div class="field" id="packagingField_${r.id}"><label>บรรจุภัณฑ์ที่ใช้</label>
    <select class="input" onchange="mapPackagingToWarehouse('${r.id}',this.value)">
      ${isUnlinked
        ? `<option value="" selected disabled>🟡 (นำเข้า) ${esc(r.unlinkedPackaging.name)} — ${num2(r.unlinkedPackaging.unitCost)} บาท/ชิ้น</option>`
        : `<option value="" ${!r.packagingId?'selected':''}>— ไม่ใช้บรรจุภัณฑ์ —</option>`}
      ${state.packaging.map(p=>`<option value="${p.id}" ${!isUnlinked && p.id===r.packagingId?'selected':''}>${esc(p.name)} (${packagingUnitCost(p).toFixed(2)} บาท/ชิ้น)</option>`).join('')}
    </select>
    ${isUnlinked ? `<button class="btn btn-ghost btn-sm" style="align-self:flex-start;padding:2px 4px;font-size:11px" onclick="addUnlinkedPackagingToWarehouse('${r.id}')">+ เพิ่มเข้าคลังบรรจุภัณฑ์</button>` : ''}
  </div>`;
}
function mapPackagingToWarehouse(rid, packagingId){
  const r = findRecipe(rid);
  r.packagingId = packagingId;
  r.unlinkedPackaging = null;
  document.getElementById('packagingField_'+rid).outerHTML = packagingFieldHTML(r);
  refreshCalcPanel(rid);
  refreshUnmappedBanner(rid);
  schedulePersist();
}
function addUnlinkedPackagingToWarehouse(rid){
  const r = findRecipe(rid);
  const snap = r.unlinkedPackaging;
  const newItem = { id: uid('pk'), name: snap.name, buyPrice: Number(snap.unitCost)||0, buyQty: 1, buyUnit: 'ชิ้น', piecesPerBox: 1 };
  state.packaging.push(newItem);
  mapPackagingToWarehouse(rid, newItem.id);
}
```

- [ ] **Step 3: Verify syntax**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 4: Browser-verify**

Recipe with unlinked packaging — check the pinned option and calc panel cost, then map it to an existing warehouse item:
```js
const pkgItem = {id:uid('pk'), name:'แก้วกาแฟ', buyPrice:1000, buyQty:200, buyUnit:'ชิ้น', piecesPerBox:100};
state.packaging.push(pkgItem);
const r = {id:uid('r'), name:'ทดสอบแพ็กเกจจิ้ง', category:'ขนม', yieldQty:10, yieldUnit:'ชิ้น',
  ingredients:[], packagingId:'', unlinkedPackaging:{name:'แก้วกาแฟ', unitCost:5},
  laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, sellPrice:0, note:''};
state.recipes.unshift(r);
openRecipe(r.id);
JSON.stringify({
  bannerMentionsPackaging: document.getElementById('unmappedBanner_'+r.id).textContent.includes('บรรจุภัณฑ์'),
  pinnedOptionShown: document.querySelector('#packagingField_'+r.id+' select').selectedOptions[0].textContent.includes('🟡'),
  packCostInCalcPanel: document.getElementById('cp_pack_'+r.id).textContent,
});
```
Expected: `{"bannerMentionsPackaging":true,"pinnedOptionShown":true,"packCostInCalcPanel":"50.00 บาท"}` (5 บาท/ชิ้น × 10 yieldQty).

```js
mapPackagingToWarehouse(r.id, pkgItem.id);
JSON.stringify({
  packagingIdSet: r.packagingId === pkgItem.id,
  unlinkedCleared: r.unlinkedPackaging === null,
  bannerGone: document.getElementById('unmappedBanner_'+r.id).innerHTML.trim() === '',
  selectNowShowsRealItem: document.querySelector('#packagingField_'+r.id+' select').selectedOptions[0].textContent.includes('แก้วกาแฟ'),
});
```
Expected: `{"packagingIdSet":true,"unlinkedCleared":true,"bannerGone":true,"selectNowShowsRealItem":true}`.

Now test "add to warehouse" from a fresh unlinked snapshot:
```js
r.unlinkedPackaging = {name:'กล่องเบเกอรี่', unitCost:8};
r.packagingId = '';
openRecipe(r.id);
const packagingCountBefore = state.packaging.length;
addUnlinkedPackagingToWarehouse(r.id);
JSON.stringify({
  packagingCountIncreased: state.packaging.length === packagingCountBefore + 1,
  newItemName: state.packaging[state.packaging.length-1].name,
  newItemPrice: state.packaging[state.packaging.length-1].buyPrice,
  recipeLinkedToNewItem: r.packagingId === state.packaging[state.packaging.length-1].id,
});
```
Expected: `{"packagingCountIncreased":true,"newItemName":"กล่องเบเกอรี่","newItemPrice":8,"recipeLinkedToNewItem":true}`.

Finally, confirm the untouched "no packaging" case still renders exactly as before (regression check for the design spec's core requirement):
```js
const r2 = {id:uid('r'), name:'จานเปล่า', category:'อาหาร', yieldQty:1, yieldUnit:'จาน',
  ingredients:[], packagingId:'', unlinkedPackaging:null,
  laborRate:0, laborHours:0, gasPct:0, deliveryCost:0, overheadPct:0, targetPct:0, sellPrice:0, note:''};
state.recipes.unshift(r2);
openRecipe(r2.id);
JSON.stringify({
  noBanner: document.getElementById('unmappedBanner_'+r2.id).innerHTML.trim() === '',
  selectShowsNoPackagingOption: document.querySelector('#packagingField_'+r2.id+' select').selectedOptions[0].textContent.includes('ไม่ใช้บรรจุภัณฑ์'),
});
```
Expected: `{"noBanner":true,"selectShowsNoPackagingOption":true}`.

Clean up:
```js
closeEditor();
state.recipes = state.recipes.filter(x => x.id !== r.id && x.id !== r2.id);
state.packaging = state.packaging.filter(p => p.name !== 'แก้วกาแฟ' && p.name !== 'กล่องเบเกอรี่');
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add packaging mapping to the recipe editor

Mirrors the Task 9 ingredient mapping pattern: the existing
"บรรจุภัณฑ์ที่ใช้" dropdown gets a pinned, disabled "🟡 (นำเข้า) ..."
option when a recipe has a pending unlinkedPackaging snapshot, picking
a real item from that same dropdown maps it, and a
"+ เพิ่มเข้าคลังบรรจุภัณฑ์" quick action creates a new warehouse item
from the snapshot and links it. Verified the untouched
"— ไม่ใช้บรรจุภัณฑ์ —" case still renders exactly as before — no
banner, no pinned option — confirming "no packaging" and "unmapped
packaging" stay two distinct states.
EOF
)"
```

---

## Task 11: End-to-end walkthrough

This task has no new code — it's a final, from-scratch manual verification pass covering the whole feature together, the way a real user would hit it, plus one more explicit round-trip check for the "no packaging" case called out by the design spec.

**Files:** none modified.

- [ ] **Step 1: Fresh load and full syntax check**

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
process.stdout.write(m[1]);
" > /tmp/fcc_check.js && node --check /tmp/fcc_check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`.

- [ ] **Step 2: Re-run every automated test from Tasks 1-2**

```bash
node /tmp/fcc_codec_test.js index.html
node /tmp/fcc_envelope_test.js index.html
```
Expected: both print their `ALL ... TESTS PASSED` line.

- [ ] **Step 3: Browser walkthrough — build a small dataset**

Navigate to `index.html` fresh (or reuse a running instance with the earlier tasks' cleanup already applied — those steps leave `state` clean). Then:

```js
const butter = {id:uid('p'), name:'เนยเค็ม', category:PANTRY_CATS[0], family:'mass', buyPrice:90, buyQty:1000, buyUnit:'กรัม'};
state.pantry.push(butter);
const cup = {id:uid('pk'), name:'แก้วกาแฟ', buyPrice:1000, buyQty:200, buyUnit:'ชิ้น', piecesPerBox:100};
state.packaging.push(cup);

newRecipe(); // brownie-with-packaging
const brownieId = state.recipes[0].id;
updRecipeField(brownieId, 'name', 'บราวนี่');
addIngredient(brownieId);
updIngField(brownieId, 0, 'pantryId', butter.id);
updIngField(brownieId, 0, 'qty', 200);
mapPackagingToWarehouse(brownieId, cup.id); // reuses the mapping function to set packagingId directly
closeEditor();

newRecipe(); // dish served directly, no packaging
const dishId = state.recipes[0].id;
updRecipeField(dishId, 'name', 'ข้าวผัด');
closeEditor();

JSON.stringify({recipeCount: state.recipes.length});
```
Expected: `{"recipeCount":2}`.

- [ ] **Step 4: Share both recipes as one bundle, decode, and inspect**

```js
const code = encodeShareCode(buildSharePayload([brownieId, dishId]));
const decoded = decodeShareCode(code);
JSON.stringify({
  ok: decoded.ok,
  recipeNames: decoded.payload.recipes.map(r=>r.name),
  brownieHasPackaging: decoded.payload.recipes.find(r=>r.name==='บราวนี่').packagingKey !== null,
  dishHasNoPackaging: decoded.payload.recipes.find(r=>r.name==='ข้าวผัด').packagingKey === null,
});
```
Expected: `{"ok":true,"recipeNames":["บราวนี่","ข้าวผัด"],"brownieHasPackaging":true,"dishHasNoPackaging":true}`.

- [ ] **Step 5: Simulate "sending to a friend" — clear the pantry/packaging (not the recipes), then import**

```js
state.pantry = [];
state.packaging = [];
const before = state.recipes.length;
const imported = applyImportedBundle(decoded.payload);
JSON.stringify({
  addedTwo: state.recipes.length === before + 2,
  brownieNeedsMapping: recipeNeedsMapping(imported.find(r=>r.name==='บราวนี่')),
  dishNeedsMapping: recipeNeedsMapping(imported.find(r=>r.name==='ข้าวผัด')),
});
```
Expected: `{"addedTwo":true,"brownieNeedsMapping":true,"dishNeedsMapping":false}` — this is the headline assertion for the whole feature: the recipe that used packaging needs attention after import, and the recipe that never used packaging needs none, exactly matching the design spec.

- [ ] **Step 6: Resolve the imported brownie's unlinked ingredient and packaging through the UI functions, confirm the badge clears**

```js
const importedBrownie = imported.find(r=>r.name==='บราวนี่');
openRecipe(importedBrownie.id);
addUnlinkedIngredientToPantry(importedBrownie.id, 0);
addUnlinkedPackagingToWarehouse(importedBrownie.id);
JSON.stringify({
  stillNeedsMapping: recipeNeedsMapping(importedBrownie),
  bannerEmpty: document.getElementById('unmappedBanner_'+importedBrownie.id).innerHTML.trim() === '',
});
```
Expected: `{"stillNeedsMapping":false,"bannerEmpty":true}`.

```js
closeEditor();
render();
document.querySelector('.recipe-card').textContent.includes('ต้องเชื่อมวัตถุดิบ');
```
Expected: `false` (the now-fully-mapped brownie, freshest in the list, no longer shows the badge — confirms Task 6/7's badge and Task 9/10's mapping actually connect end to end through a real render, not just isolated function calls).

- [ ] **Step 7: Clean up the test data**

```js
state.recipes = state.recipes.filter(r => !['บราวนี่','ข้าวผัด'].includes(r.name));
state.pantry = state.pantry.filter(p => p.name !== 'เนยเค็ม');
state.packaging = state.packaging.filter(p => p.name !== 'แก้วกาแฟ');
schedulePersist();
'cleaned up';
```
Expected: `"cleaned up"`.

- [ ] **Step 8: Commit** (only if Step 6/7 uncovered and fixed anything — otherwise this task produces no diff and can be skipped)

If a fix was needed:
```bash
git add index.html
git commit -m "fix: <describe what the end-to-end walkthrough caught>"
```

---
