# Running YiHuo ETor on Linux / Steam Deck (Proton & Wine)

> **Status:** exploratory community note — **not a fix**, and **not** from the
> original developers. It documents a rendering problem seen when the Windows
> build is run under Wine, why it happens, and two ways to work around it: a
> deterministic recipe an AI agent can execute, and a copy-paste block a person can
> apply by hand. Offered only because Linux/Steam Deck users hit a complete (but
> easily worked-around) breakage. Feel free to close, move, or reword this.

## Symptom

On Linux under Wine (including Proton / Steam Deck), the app window opens and the
sidebar, icons, borders and numeric widgets all draw — **but no text is visible**,
so the UI cannot be navigated.

## Cause

The remote web UI (`https://etor-beta.710421059.xyz`) sets, roughly:

```css
font-family: "Segoe UI", "Microsoft YaHei", sans-serif;
```

On Windows both families exist. Under Wine they do not, **and** Wine's
DirectWrite/Chromium backend resolves the *generic* `sans-serif` family to **zero
advance width**. Every glyph therefore measures `0px`, so Blink lays text out with
no width and nothing paints. SVG icons are unaffected, which is why the window is
not completely blank.

Reproducible check in the renderer's DevTools console:

```js
const c = document.createElement('canvas').getContext('2d');
c.font = '16px sans-serif';   c.measureText('MMMM').width;  // 0   <- the bug
c.font = '16px "Segoe UI"';   c.measureText('MMMM').width;  // 0
c.font = '16px "Noto Sans"';  c.measureText('MMMM').width;  // >0  <- works
c.font = '16px system-ui';    c.measureText('MMMM').width;  // >0  <- works
```

## Workaround (user side — no app changes)

Teach Wine that `"Segoe UI"` / `"Microsoft YaHei"` exist by installing fonts that
carry those exact family names (aliases of Noto Sans / Noto Sans CJK), then refresh
Wine's font database. This is entirely user-side, touches neither the app nor its
files, and is reversible. It also repairs the generic `sans-serif` mapping for
other Wine apps in that prefix.

Two ways to do it: a deterministic recipe for an agent (next), or a single
copy-paste block for a person (after that).

## Agent replication (deterministic)

End state: the renderer measures text with **non-zero** width and
`document.body.innerText` is populated. Steps are idempotent; verify after each.

### 0. Discover inputs

```bash
python3 -c 'import fontTools; print("fontTools", fontTools.version)'   # else: python3 -m pip install --user fonttools
fc-match -f '%{file}\n' 'Noto Sans'          # -> .../NotoSans-Regular.ttf
fc-match -f '%{file}\n' 'Noto Sans CJK SC'   # -> .../NotoSansCJK-Regular.ttc

# Find the ETOR Wine prefix (the directory that contains drive_c):
find "$HOME" -maxdepth 6 -type d -name drive_c 2>/dev/null
```

### 1. Create the alias fonts

```bash
ALIAS="$HOME/.local/share/fonts/etor-alias"
mkdir -p "$ALIAS"
python3 - "$ALIAS" <<'PY'
import os, sys
from fontTools.ttLib import TTFont, TTCollection

OUT = sys.argv[1]
os.makedirs(OUT, exist_ok=True)

def rename(font, family, subfamily, psname, fullname, weight):
    name = font["name"]
    for rec in list(name.names):
        if rec.nameID in (1, 2, 3, 4, 6, 16, 17):
            name.removeNames(rec.nameID, rec.platformID, rec.platEncID, rec.langID)
    def add(nid, s):
        name.setName(s, nid, 3, 1, 0x409)   # Windows / en-US
        name.setName(s, nid, 1, 0, 0)       # Mac / Roman
    add(1, family); add(2, subfamily); add(3, f"{family}-{subfamily}")
    add(4, fullname); add(6, psname); add(16, family); add(17, subfamily)
    font["OS/2"].usWeightClass = weight
    if "Bold" in subfamily:
        font["OS/2"].fsSelection |= (1 << 5)
    else:
        font["OS/2"].fsSelection &= ~(1 << 5)

# Latin glyphs -> "Segoe UI"
for src, sub, ps, wt in [
    ("/usr/share/fonts/noto/NotoSans-Regular.ttf", "Regular", "SegoeUI",      400),
    ("/usr/share/fonts/noto/NotoSans-Bold.ttf",    "Bold",    "SegoeUI-Bold", 700),
]:
    f = TTFont(src)
    rename(f, "Segoe UI", sub, ps,
           "Segoe UI" if sub == "Regular" else f"Segoe UI {sub}", wt)
    f.save(os.path.join(OUT, ps + ".ttf"))

# CJK glyphs -> "Microsoft YaHei" (pull the SC face out of each TTC)
for ttc, bold in [("/usr/share/fonts/noto-cjk/NotoSansCJK-Regular.ttc", False),
                  ("/usr/share/fonts/noto-cjk/NotoSansCJK-Bold.ttc",    True)]:
    coll = TTCollection(ttc)
    face = next(f for f in coll.fonts
                if "CJK SC" in (f["name"].getDebugName(1) or "")
                and "Mono" not in (f["name"].getDebugName(1) or ""))
    sub = "Bold" if bold else "Regular"
    ps = "MicrosoftYaHei-Bold" if bold else "MicrosoftYaHei"
    rename(face, "Microsoft YaHei", sub, ps,
           "Microsoft YaHei Bold" if bold else "Microsoft YaHei",
           700 if bold else 400)
    face.save(os.path.join(OUT, ps + ".otf"))
PY
```

### 2. Register with fontconfig and Wine

```bash
ALIAS="$HOME/.local/share/fonts/etor-alias"
fc-cache -f "$ALIAS"
fc-match 'Segoe UI'          # must print a path under $ALIAS
fc-match 'Microsoft YaHei'   # must print a path under $ALIAS

export WINEPREFIX="<the ETOR prefix from step 0>"
export WINEARCH=win64 WINEDEBUG=-all
wineserver -k 2>/dev/null || true; sleep 1
wineboot -u
wine reg query 'HKLM\Software\Microsoft\Windows NT\CurrentVersion\Fonts' | grep -i 'Segoe UI'
```

> **Do not** drop the fonts into `$WINEPREFIX/drive_c/windows/Fonts/`. Wine does
> not register fonts from there in this configuration; host fontconfig plus a
> `wineboot -u` refresh is what works (verified).

### 3. Launch with a debug port

```bash
APP="<directory containing the *-ETor.exe>"
EXE="$(find "$APP" -maxdepth 1 -iname '*-ETor.exe' -printf '%f\n' | head -1)"
cd "$APP"
setsid nohup wine "$EXE" --no-sandbox --disable-gpu \
  --remote-debugging-port=9222 --remote-allow-origins='*' >/tmp/etor.log 2>&1 &
sleep 12
```

### 4. Verify over CDP

Get the page target:

```bash
WS=$(curl -s http://127.0.0.1:9222/json/list \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)[0]["webSocketDebuggerUrl"])')
```

Then `Runtime.evaluate` (any CDP client) with:

```js
(() => {
  const c = document.createElement('canvas').getContext('2d');
  const w = f => { c.font = '16px ' + f; return Math.round(c.measureText('MMMM').width * 100) / 100; };
  return JSON.stringify({
    sans:   w('sans-serif'),
    segoe:  w('"Segoe UI"'),
    yahei:  w('"Microsoft YaHei"'),
    textLen: document.body.innerText.length,
    sample: document.body.innerText.slice(0, 120)
  });
})()
```

Success criteria:

| key | broken | fixed |
| --- | --- | --- |
| `sans` | `0` | `~48–58` |
| `segoe` | `0` | `~58` |
| `yahei` | `0` | `~52` |
| `textLen` | `~32` (digits only) | hundreds (labels present) |

If `sans` is still `0`, the Wine font registry did not refresh: re-run
`wineserver -k` + `wineboot -u` in the correct `WINEPREFIX`, then relaunch.

### 5. Rollback

```bash
rm -rf "$HOME/.local/share/fonts/etor-alias" && fc-cache -f
```

## Manual application (no agent)

If you'd rather just do it yourself: paste the block below into a terminal. It does
the whole thing — creates fonts named `"Segoe UI"` / `"Microsoft YaHei"` (copies of
Noto Sans / Noto Sans CJK), tells fontconfig, and refreshes the Wine font database.

**Before you start** you need `python3` with `fontTools`, and the Noto fonts. With
your distro's package manager (examples; names vary):

```bash
python3 -m pip install --user fonttools        # or python-fonttools from your repos
# Arch/CachyOS:   sudo pacman -S noto-fonts noto-fonts-cjk
# Debian/Ubuntu:  sudo apt install fonts-noto fonts-noto-cjk python3-fonttools
# Fedora:         sudo dnf install google-noto-sans-fonts google-noto-sans-cjk-fonts python3-fonttools
```

Find ETOR's Wine prefix — the folder that **contains `drive_c`**:

```bash
find "$HOME" -maxdepth 6 -type d -name drive_c 2>/dev/null
```

Then save and run this, editing `WINEPREFIX` on the first line:

```bash
#!/usr/bin/env bash
WINEPREFIX="${WINEPREFIX:-$HOME/.wine}"     # <-- EDIT to the ETOR prefix (contains drive_c)
ALIAS="$HOME/.local/share/fonts/etor-alias"
mkdir -p "$ALIAS"

python3 - "$ALIAS" <<'PY'
import os, sys
from fontTools.ttLib import TTFont, TTCollection

OUT = sys.argv[1]
os.makedirs(OUT, exist_ok=True)

def rename(font, family, subfamily, psname, fullname, weight):
    name = font["name"]
    for rec in list(name.names):
        if rec.nameID in (1, 2, 3, 4, 6, 16, 17):
            name.removeNames(rec.nameID, rec.platformID, rec.platEncID, rec.langID)
    def add(nid, s):
        name.setName(s, nid, 3, 1, 0x409)
        name.setName(s, nid, 1, 0, 0)
    add(1, family); add(2, subfamily); add(3, f"{family}-{subfamily}")
    add(4, fullname); add(6, psname); add(16, family); add(17, subfamily)
    font["OS/2"].usWeightClass = weight
    if "Bold" in subfamily:
        font["OS/2"].fsSelection |= (1 << 5)
    else:
        font["OS/2"].fsSelection &= ~(1 << 5)

for src, sub, ps, wt in [
    ("/usr/share/fonts/noto/NotoSans-Regular.ttf", "Regular", "SegoeUI",      400),
    ("/usr/share/fonts/noto/NotoSans-Bold.ttf",    "Bold",    "SegoeUI-Bold", 700),
]:
    f = TTFont(src)
    rename(f, "Segoe UI", sub, ps,
           "Segoe UI" if sub == "Regular" else f"Segoe UI {sub}", wt)
    f.save(os.path.join(OUT, ps + ".ttf"))

for ttc, bold in [("/usr/share/fonts/noto-cjk/NotoSansCJK-Regular.ttc", False),
                  ("/usr/share/fonts/noto-cjk/NotoSansCJK-Bold.ttc",    True)]:
    coll = TTCollection(ttc)
    face = next(f for f in coll.fonts
                if "CJK SC" in (f["name"].getDebugName(1) or "")
                and "Mono" not in (f["name"].getDebugName(1) or ""))
    sub = "Bold" if bold else "Regular"
    ps = "MicrosoftYaHei-Bold" if bold else "MicrosoftYaHei"
    rename(face, "Microsoft YaHei", sub, ps,
           "Microsoft YaHei Bold" if bold else "Microsoft YaHei",
           700 if bold else 400)
    face.save(os.path.join(OUT, ps + ".otf"))
PY

fc-cache -f "$ALIAS"
wineserver -k 2>/dev/null || true
WINEPREFIX="$WINEPREFIX" WINEARCH=win64 WINEDEBUG=-all wineboot -u
echo "Done. Relaunch ETOR. To undo: rm -rf \"$ALIAS\" && fc-cache -f"
```

Then launch ETOR normally. **What you should see:** before the fix, the window shows
numbers/icons but no words; after, it shows `Overview`, `Items`, `Price Database`,
`History`, and so on.

**If text is still missing:**

* You refreshed the wrong prefix — it must be the folder containing `drive_c`.
* You copied fonts into `drive_c/windows/Fonts/` — that does **not** work here; use
  the script above.
* A Wine process from before is still running: `wineserver -k`, then relaunch.

**Undo:** `rm -rf ~/.local/share/fonts/etor-alias && fc-cache -f`

## Possible app-side alternative (only actionable by whoever controls the UI)

Adding Linux-available fallbacks to the font stack would fix it with no OS changes:

```css
font-family: "Segoe UI", "Microsoft YaHei", "Noto Sans", "Noto Sans CJK SC",
             system-ui, sans-serif;
```

Both `"Noto Sans"` and `system-ui` measure non-zero under Wine (verified). The CSS
itself lives in the **remote** app, so it can only be changed by the original
authors. That said, the translation layer here *is* injected into every page
(`build/obf-app/translate.js`, loaded via `mainWindow.webContents.executeJavaScript`
in `main.js`), so a maintainer could ship the fallback locally by prepending:

```js
const s = document.createElement('style');
s.textContent = 'html,body,button,input,select,textarea{font-family:' +
  '"Segoe UI","Microsoft YaHei","Noto Sans","Noto Sans CJK SC",system-ui,sans-serif !important}';
(document.head || document.documentElement).appendChild(s);
```

Caveat: that means repacking `resources/app.asar`, and the override would need to be
re-checked whenever the remote styling changes.

## Recommendation

Best raised with the original YiHuo ETor developers — as `README.md` already notes,
core-functionality feedback goes to them. In the meantime, the font-alias workaround
above restores the UI for affected users.
