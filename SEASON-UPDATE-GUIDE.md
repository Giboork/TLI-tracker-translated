# Season Update Runbook — Translating a New ETor Version

How to produce a new English translation release (e.g. `v1.2.0`) when the game/app ships a new
season. Written from the v1.2.0 update. Follow top to bottom.

---

## 0. How the translation actually works

ETor (易火-ETor) is an Electron shell that loads a **remote** web app
(`https://etor-beta.710421059.xyz/?t=<timestamp>`, see `build/obf-app/main.js`). The UI text is
Chinese. We don't modify the web app — we **inject a translation layer** locally:

- `build/obf-app/translations.json` — the dictionary: `{ "中文": "English" }`, grouped into
  sections. **All sections are flattened into one flat map at runtime**, so section names are just
  for organization — a key in any section is looked up globally.
- `build/obf-app/translate.js` — injected into every page. For each on-screen text node it:
  1. tries an **exact match** in the dictionary,
  2. applies hard-coded **regex patterns** for dynamic strings (counts, dates, `{placeholder}`
     templates),
  3. falls back to a **substring-replacement loop** over every dictionary entry
     (longest key first, regex-escaped — see §9).
- Both files live inside `resources/app.asar`. `main.js` copies the bundled `translations.json`
  into `%APPDATA%\etor\translations.json` on **every launch**, then injects it. So the **source of
  truth is the copy inside `app.asar`** — editing the AppData copy does nothing (it's overwritten
  each start).

**Consequence:** to change translations you must repack `app.asar`. There is no hot reload.

The editable/extracted copy of the app lives in `extracted-app/` (git-ignored). Keep it in sync
with what you pack.

---

## 1. Prerequisites

- **Node.js** (ships in `extracted-app/node_modules`, or system node)
- **@electron/asar** CLI: `npx asar` (v3.2.0 used)
- **curl** with brotli support (`--compressed`)
- **gh** CLI, authenticated to the `Giboork` account
- A scratch dir to work in

---

## 2. Fetch the new bundles (past the WAF)

The app's CDN (TencentEdgeOne) returns **HTTP 567** (a JS anti-bot challenge, a ~7 KB spinner page)
to plain `curl` — it fingerprints non-browser clients. **Real Chrome headers + a `Referer` +
`Sec-Fetch-*` get through.** Grab the current asset hashes from the running app's DevTools (Network
tab) — they change every build.

```bash
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36"
for f in index-XXXX.js vendor_vue-XXXX.js vendor_libs-XXXX.js game_static_data-XXXX.js; do
  curl -sS --compressed "https://etor-beta.710421059.xyz/assets/$f" \
    -H "User-Agent: $UA" -H "Accept: */*" -H "Accept-Language: zh-CN,zh;q=0.9,en;q=0.8" \
    -H "Referer: https://etor-beta.710421059.xyz/" \
    -H "Sec-Fetch-Dest: script" -H "Sec-Fetch-Mode: no-cors" -H "Sec-Fetch-Site: same-origin" \
    -o "$f" -w "$f -> HTTP %{http_code} | %{size_download} bytes\n"
done
```

A real bundle is hundreds of KB. If you get ~7 KB back, the headers/hashes are wrong.
Only `index-*` and `game_static_data-*` carry Chinese; the `vendor_*` bundles are libraries.

> Fallback if the WAF ever changes: the running app has already downloaded the bundles — paste the
> DevTools-console snippet in `main.js` land that `fetch()`es each `performance.getEntriesByType('resource')`
> JS asset and triggers a download.

---

## 3. Extract the base dictionary from the shipped asar

The repo's `extracted-app/build/obf-app/translations.json` can be **stale**. The real base is inside
the current `resources/app.asar`:

```bash
npx asar extract resources/app.asar asar-work
# base dict:  asar-work/build/obf-app/translations.json
# matcher:    asar-work/build/obf-app/translate.js
```

> The asar stores paths with **backslashes**, so `asar extract-file "build/obf-app/..."` (forward
> slashes) fails silently — always `asar extract` the whole thing.

---

## 4. Extract Chinese strings from the new bundles

`game_static_data` is a big `JSON.parse('[...]')` array of objects with fields
`{name, type, img, id, reward, url, baseItemId}`. **The `url` field is gold: its last path segment
is the canonical tlidb English slug** (e.g. `崩山击` → `https://tlidb.com/cn/Earthshatter` →
`Earthshatter`). The icons also point at `cdn.tlidb.com` with the same IDs.

`extract2.js` — pull clean strings, filtering leaked source code:

```js
const fs = require("fs");
const CJK = /[一-鿿]/;
function parseAll(src){const out=[];const re=/JSON\.parse\('((?:\\.|[^\\'])*)'\)/g;let m;
  while((m=re.exec(src))){try{out.push(JSON.parse(m[1].replace(/\\'/g,"'").replace(/\\\\/g,"\\")));}catch(e){}}return out;}
const isCode = s => /[\$][{]|=>|<\/|<[a-zA-Z][^>]*>|class=|\.value|function |return |;const |\bvar \b|\`/.test(s) || /\n/.test(s);

// game_static_data: name/type/reward + short CJK field values
const gsrc = fs.readFileSync("game_static_data-XXXX.js","utf8");
let objs=[]; for(const a of parseAll(gsrc)) if(Array.isArray(a)) objs=objs.concat(a.filter(x=>x&&typeof x==="object"));
const game=new Set(), nameToEn=new Map();
const TEXT=new Set(["name","type","reward","desc","title","subType"]);
for(const o of objs){
  if(typeof o.name==="string" && o.url){ const en=decodeURIComponent(o.url.split("/").filter(Boolean).pop()).replace(/_/g," ").trim();
    if(/[A-Za-z]/.test(en)&&!/[<>]/.test(en)) nameToEn.set(o.name,en); }
  for(const k in o){ const v=o[k];
    if(typeof v==="string"&&CJK.test(v)&&!isCode(v)&&(TEXT.has(k)||v.length<=60)) game.add(v); }
}
// index bundle: quoted literals, filtered
function lits(file){const src=fs.readFileSync(file,"utf8");const set=new Set();
  const re=/"((?:[^"\\]|\\.)*)"|'((?:[^'\\]|\\.)*)'|`((?:[^`\\]|\\.)*)`/g;let m;
  while((m=re.exec(src))){let s=m[1]??m[2]??m[3];if(!s)continue;
    s=s.replace(/\\u([0-9a-fA-F]{4})/g,(_,h)=>String.fromCharCode(parseInt(h,16)));
    if(CJK.test(s)&&s.length<120&&!isCode(s))set.add(s);}return set;}
const index=lits("index-XXXX.js");
fs.writeFileSync("game-strings.json",JSON.stringify([...game].sort(),null,1));
fs.writeFileSync("index-strings.json",JSON.stringify([...index].sort(),null,1));
fs.writeFileSync("nameToEn.json",JSON.stringify(Object.fromEntries(nameToEn),null,1));
console.log("game:",game.size,"index:",index.size,"tlidb slug map:",nameToEn.size);
```

---

## 5. Compute what's actually new (diff against base)

```js
const fs=require("fs");
const flat=(o,m)=>{for(const k in o){const v=o[k];v&&typeof v==="object"?flat(v,m):m.set(k,v);}return m;};
const base=flat(JSON.parse(fs.readFileSync("asar-work/build/obf-app/translations.json","utf8")),new Map());
const slug=new Map(Object.entries(JSON.parse(fs.readFileSync("nameToEn.json","utf8"))));
const strings=[...JSON.parse(fs.readFileSync("game-strings.json")),...JSON.parse(fs.readFileSync("index-strings.json"))];
const newFromSlug={}, manual=[];
for(const s of new Set(strings)){
  if(base.has(s)) continue;                        // already translated
  if(slug.has(s)){ newFromSlug[s]=slug.get(s); continue; } // authoritative, free
  manual.push(s);                                  // needs translation
}
fs.writeFileSync("new-from-slug.json",JSON.stringify(newFromSlug,null,1));
fs.writeFileSync("manual.json",JSON.stringify(manual.sort((a,b)=>a.length-b.length),null,1));
console.log("auto (tlidb slug):",Object.keys(newFromSlug).length,"| manual:",manual.length);
```

Split `manual.json` into: **placeholder templates** (contain `{name}`/`{count}` → need regex patterns,
§7), **tiny 1–3 char** (risky, §7), and the rest (bulk UI, §6).

---

## 6. Translate the bulk (glossary-consistent)

Build a glossary so terms match what's already shipped, then translate. In v1.2 the 620 UI strings
were split into batches and translated by parallel sub-agents, each given the glossary; you can also
do it by hand.

**Non-negotiable glossary rules** (from the existing dict — always confirm against it):
`罗盘=Compass 信标=Beacon 异界=Netherrealm 叠界=Overrealm 囚笼=Cage 巨力=Strength 黑潮=Dark Tide
首领=Boss 灰烬=Ember 塔罗=Tarot 探针=Probe 赛季=Season 策略=Strategy 收益=Profit 天赋=Talent
扈从=followers 宝箱=chest 神骸=Divine Remnant 明月=Lunaria 渴瘾=Vorax 怪谈=Folklore 火=FE`

For game nouns (items/skills/affixes) prefer `nameToEn.json` (tlidb) over guessing.
Output rules: strict JSON `{zh:en}`, every input key present, **no Chinese left in values**,
convert full-width punctuation (`，。：（）！～` → `,.:()!~`), preserve numbers/`%`/`🔥`/`×`.

---

## 7. Placeholder patterns + tiny keys (do these by hand)

**Placeholder templates** like `共 {count} 条` render at runtime as `共 5 条`, so an exact key never
matches — add a regex to `translate.js` (before the substring loop) instead:

```js
result = result.replace(/共 (\d+) 条/g, 'Total $1 records');
result = result.replace(/赛季 (\d+)/g, 'Season $1');
result = result.replace(/上次：(.+)$/g, 'Last: $1');
```

**Tiny 1–3 char keys** (中/无/天/火…) are fine as exact keys but dangerous in the substring loop —
they can appear inside longer words. The matcher hardening in §9 (longest-first + escaping) makes
them safe; still, translate them precisely (they were a theme/color picker + status words in v1.2).

---

## 8. Merge + validate

```js
const fs=require("fs");
const base=JSON.parse(fs.readFileSync("asar-work/build/obf-app/translations.json","utf8"));
const merged={...base,
  season_items:JSON.parse(fs.readFileSync("new-from-slug.json")),
  season_ui:Object.assign({}, /* out-batch1..N.json */),
  season_tiny:JSON.parse(fs.readFileSync("out-tiny.json")),
  season_game:JSON.parse(fs.readFileSync("out-game.json"))};
// VALIDATE: no empty values, no Chinese left in values, no key collisions, no base overrides
const CJK=/[一-鿿]/; const flat=(o,m)=>{for(const k in o){const v=o[k];v&&typeof v==="object"?flat(v,m):m.set(k,v);}return m;};
// ...assert clean, then write:
const json=JSON.stringify(merged,null,2);
for(const t of ["asar-work/build/obf-app/translations.json","asar-work/translations.json",
                "extracted-app/build/obf-app/translations.json","extracted-app/translations.json"])
  fs.writeFileSync(t,json,"utf8");
```

Always run the validator — a stray Chinese character or a key collision is the usual bug.

---

## 9. Matcher hardening (already applied — keep it)

`translate.js` was patched once (v1.2) and should stay this way. After the flatten block:

```js
const translationEntries = Object.entries(translations)
  .filter(([zh]) => zh)
  .sort((a, b) => b[0].length - a[0].length)                       // longest key first
  .map(([zh, en]) => [new RegExp(zh.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'g'), en, zh]); // escaped
```

and the substring loop uses it:

```js
for (const [re, english, chinese] of translationEntries) {
  if (result.includes(chinese)) { result = result.replace(re, english); hadPartialMatch = true; }
}
```

Without this, keys containing regex metacharacters (affix sentences with `()`/`.`/`%`) crash the loop,
and short keys clobber longer phrases.

---

## 10. Repack + local test

`app.asar` is **locked while ETor runs** — fully quit it first (check Task Manager for
`易火-ETor.exe`).

```bash
cp resources/app.asar resources/../app.asar.vPREV.bak     # keep OUT of resources/ so it isn't zipped
npx asar pack asar-work new-app.asar
# verify entry count before swapping:
npx asar extract new-app.asar verify && node -e 'c=0;(function f(o){for(k in o){v=o[k];v&&typeof v=="object"?f(v):c++}})(require("./verify/build/obf-app/translations.json"));console.log(c)'
cp new-app.asar resources/app.asar
```

Launch `易火-ETor.exe` and eyeball: new item/skill names, affix effect text, season UI, counters
like "Day 5 / Week 2". Fixes are cheap now — add the exact string and repack.

---

## 11. Release

Convention: each release is a single commit of `resources/app.asar`, plus a GitHub Release with the
zip. Internal `package.json` version is **not** bumped (stays the original app version).

```bash
# 1. bump the zip version in create-release.bat (v1.1.0 -> v1.2.0)
# 2. commit the new asar
git add resources/app.asar
git commit -m "v1.2.0: <summary>"     # NO AI attribution
git push

# 3. build the zip (create-release.bat, or the PowerShell equivalent — copies exe/dll/pak/data +
#    resources/ + locales/ into a staging dir and Compress-Archives it). ~115 MB.

# 4. GitHub Release (draft first so you can confirm the season name in the title, then Publish):
gh release create v1.2.0 --draft --target main \
  --title "v1.2.0 - <Season> Translations" \
  --notes "..." \
  TLI-tracker-translated-v1.2.0.zip
```

---

## Gotchas cheat-sheet

- **WAF 567** on curl → add Chrome UA + `Referer` + `Sec-Fetch-*` + `--compressed`.
- **asar internal paths use backslashes** → use `asar extract` (whole), not `extract-file`.
- **AppData copy is overwritten every launch** → only the copy inside `app.asar` matters.
- **app.asar is locked while running** → quit ETor before swapping.
- **tlidb URL slug loses apostrophes** (`Prism's Echo` → `Prisms Echo`) → prefer the existing dict
  value when it already has proper punctuation; don't "correct" it to the slug.
- **Substring loop** replaces globally → keep the longest-first + regex-escape patch (§9).
- **Keep `extracted-app/` in sync** with what you pack, so the next diff is clean.
