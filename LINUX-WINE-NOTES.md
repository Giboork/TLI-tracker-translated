# Running YiHuo ETor on Linux / Steam Deck (Wine)

> **Status:** community note — **not a fix**, and **not** from the original
> developers. It is written for people who are on Linux (including Steam Deck
> users) and want to get the Windows build of YiHuo ETor running. It covers the
> whole path — install, a known “no text” rendering bug, and the one-time setup
> that makes it actually track loot — plus a deterministic recipe an AI agent can
> execute. Offered because none of this is documented anywhere else. Feel free to
> close, move, or reword this.

## What's in this file

1. **Part 1 — Install ETOR under Wine** (download, extract, prefix, launch)
2. **Part 2 — Make the text visible** (the known bug, with a copy‑paste fix)
3. **Part 3 — First‑run setup** (point ETOR at the game log so it tracks)
4. **Troubleshooting**
5. **Appendix A — for AI agents** (deterministic replication)
6. **Appendix B — a possible app‑side fix** (for whoever controls the UI)

If you only have the “window draws but no words” problem and everything else
works, jump straight to **Part 2**.

---

## Before you start

You need:

| Thing | Why | Notes |
| --- | --- | --- |
| **Wine** | runs the Windows `.exe` on Linux | tested with **Wine 11.19**. Check with `wine --version`. |
| **Torchlight: Infinite** (Steam) | the game whose log ETOR reads | ETOR runs in its **own** Wine prefix; the game can stay on Proton. |
| **An internet connection** | ETOR’s UI is a **remote web app** | `etor-beta.710421059.xyz` does all the parsing. No internet = nothing works. |
| `python3` + `fontTools` | only for the text fix (Part 2) | `python3 -m pip install --user fonttools` |
| Noto fonts | only for the text fix (Part 2) | `noto-fonts` + `noto-fonts-cjk` |

**On desktop Linux**, install Wine and the fonts with your package manager
(examples; names vary):

```bash
# Arch / CachyOS
sudo pacman -S wine noto-fonts noto-fonts-cjk
# Debian / Ubuntu
sudo apt install wine fonts-noto fonts-noto-cjk python3-fonttools
# Fedora
sudo dnf install wine google-noto-sans-fonts google-noto-sans-cjk-fonts python3-fonttools
```

**On Steam Deck**, a `wine` command is not normally available. The simplest route
is to run the `.exe` through a Wine wrapper such as **Bottles** (Flatpak) or
**Lutris**. This guide was verified with a plain system Wine, so treat the Steam
Deck wording as a starting point rather than a tested recipe.

> **Never** put ETOR in the game’s Proton prefix
> (`steamapps/compatdata/1974050/pfx`). Use its own prefix (Part 1c) so the two
> stay isolated.

---

## Part 1 — Install ETOR under Wine

### 1a. Download the release

1. Open the [Releases page](https://github.com/Giboork/TLI-tracker-translated/releases).
2. Download the newest ZIP, e.g. **`TLI-tracker-translated-v1.2.0.zip`** (~120 MB).
3. Move it somewhere easy, e.g. `~/Downloads`.

### 1b. Extract it (don’t skip the warning)

The ZIP was built on Windows, so inside it the folders are separated with
backslashes (`locales\af.pak`). Linux tools can trip on that.

```bash
mkdir -p ~/Applications/ETOR
cd ~/Applications/ETOR
unzip ~/Downloads/TLI-tracker-translated-v1.2.0.zip -d app
```

`unzip` understands the Windows separators and prints a harmless warning:

```
warning: ... appears to use backslashes as path separators
```

> **Do not use `7z` to extract this archive.** `7z` writes the entries literally,
> producing files named `locales\af.pak` and a `app/` folder that will not start.

Check it worked — you should see the `.exe`, a `locales` folder, and a `resources`
folder:

```bash
ls ~/Applications/ETOR/app
# ... 易火-ETor.exe  locales  resources  ...
```

### 1c. Create a dedicated Wine prefix

A “Wine prefix” is a private fake‑Windows folder for this app. Create one **once**:

```bash
export WINEPREFIX="$HOME/Applications/ETOR/prefix"
export WINEARCH=win64
export WINEDEBUG=-all
wineboot -u
```

The first run takes a minute and may print setup messages. When it finishes,
`~/Applications/ETOR/prefix/drive_c` exists.

### 1d. Launch it

```bash
cd ~/Applications/ETOR/app
WINEPREFIX="$HOME/Applications/ETOR/prefix" wine ./*-ETor.exe --no-sandbox --disable-gpu
```

* `--no-sandbox --disable-gpu` are Chromium flags that make Electron more reliable
  under Wine. They are safe.
* The first launch is slow, and Wine/Electron prints lots of warnings — that is
  normal. The window should appear and load the remote UI.

If the window opens but **every label is missing**, that is the bug in **Part 2**.
If it never opens at all, see **Troubleshooting**.

**Daily use:** save this as `~/Applications/ETOR/start-etor.sh` so you don’t retype
the flags (make it executable with `chmod +x`):

```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT="$HOME/Applications/ETOR"
export WINEPREFIX="$ROOT/prefix" WINEARCH=win64 WINEDEBUG="${WINEDEBUG:--all}"
cd "$ROOT/app"
setsid nohup wine ./*-ETor.exe --no-sandbox --disable-gpu >"$ROOT/etor.log" 2>&1 &
echo "ETOR starting (log: $ROOT/etor.log)"
```

---

## Part 2 — Make the text visible (known bug)

### Symptom

The window opens, and the sidebar, icons, borders and numbers are all there — but
**no words are visible**, so you cannot navigate. (This is *not* the same as a
completely empty window; see Troubleshooting for that one.)

### Why it happens

ETOR’s remote web UI sets:

```css
font-family: "Segoe UI", "Microsoft YaHei", sans-serif;
```

* `"Segoe UI"` (Windows) and `"Microsoft YaHei"` (Chinese) do not exist on Linux.
* Worse, under Wine the generic fallback `sans-serif` is reported with **zero
  width**, so every character is laid out as `0px` wide and nothing is painted.
  Icons (SVG) still show, which is why the window isn’t blank.

You can see it for yourself in the renderer’s console:
`16px sans-serif` → `measureText("MMMM").width` = **0**, while `16px "Noto Sans"`
is non‑zero.

The fix is to give Wine fonts that literally carry the names `"Segoe UI"` and
`"Microsoft YaHei"`, made by copying Noto Sans / Noto Sans CJK.

### The fix (copy‑paste)

This is **user‑level and fully reversible** — it touches neither the app nor your
game. Edit `WINEPREFIX` on the first line to your ETOR prefix (the folder that
contains `drive_c`):

```bash
#!/usr/bin/env bash
WINEPREFIX="$HOME/Applications/ETOR/prefix"     # <-- your ETOR prefix (contains drive_c)
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

fc-cache -f "$ALIAS"
wineserver -k 2>/dev/null || true
WINEPREFIX="$WINEPREFIX" WINEARCH=win64 WINEDEBUG=-all wineboot -u
echo "Done. Relaunch ETOR. To undo: rm -rf \"$ALIAS\" && fc-cache -f"
```

Then **relaunch ETOR** (Part 1d). The sidebar should now read `Overview`, `Items`,
`Price Database`, `History`, etc.

A few things worth knowing:

* `wineserver -k` + `wineboot -u` makes Wine re‑scan the system fonts. It only
  needs to happen once, but it is safe to repeat.
* **Do not** copy fonts into `drive_c/windows/Fonts/` instead — Wine does not pick
  fonts up from there in this setup; host fontconfig plus `wineboot -u` is what
  works (verified).
* **Undo:** `rm -rf ~/.local/share/fonts/etor-alias && fc-cache -f`.
* If you cannot install Python/`fontTools`, this specific fix isn’t available; ask
  someone technical to run the block for you.

---

## Part 3 — First run: point ETOR at the game log

ETOR can only track loot if the game is writing a log and ETOR knows where it is.
This is a **one‑time** setup.

### 3a. Turn on logging in the game

Logging is off by default. In Torchlight: Infinite:

**Settings → Other → enable logging**, then **restart / re‑login**.

Without this, the log contains only asset‑loading noise and neither this tool nor
any other tracker will see anything.

### 3b. Tell ETOR where the log is

ETOR does **not** auto‑detect the Steam log on Linux (the translated build is
missing the auto‑discovery call), so set it by hand once.

Find the log (adjust if your Steam library is elsewhere):

```
~/.local/share/Steam/steamapps/common/Torchlight Infinite/UE_game/TorchLight/Saved/Logs/UE_game.log
```

In **ETOR → Settings → log path**, paste the **Windows‑style** path. Because Wine
maps `Z:` to `/`, it looks like this (use *your* Linux username):

```
Z:\home\<your-user>\.local\share\Steam\steamapps\common\Torchlight Infinite\UE_game\TorchLight\Saved\Logs\UE_game.log
```

Tip: pasting just the folder also works — the app appends `\UE_game.log`.

The setting is saved in ETOR’s Electron profile
(`prefix/drive_c/users/<you>/AppData/Roaming/etor/`), so you won’t have to do it
again.

### 3c. Confirm it works

With the game running and logging on, you should see character/level and item
pickups appear in ETOR within a few seconds of picking something up. If not, check
Troubleshooting.

---

## Troubleshooting

**The window is completely empty / can’t reach anything.**
ETOR’s UI is a remote web app. Confirm you are online and that
`https://etor-beta.710421059.xyz` is reachable. This is a *different* problem from
Part 2 (where layout and numbers are visible but words are not).

**The window draws, but no words appear.**
The Part 2 font fix did not take effect. Re‑run it (make sure `WINEPREFIX` points
at the ETOR prefix that contains `drive_c`), then relaunch.

**ETOR opens but never tracks anything.**
Almost always the log path: it must be set (3b), it must be a `Z:\…` path, and
in‑game logging must be on (3a). Also make sure you’re not sitting in a hub — hubs
don’t produce runs.

**`Error occurred in handler for 'get-back-version': No handler registered`.**
Harmless / cosmetic. Ignore it.

**Wine errors, or the app won’t start.**
Start it from a terminal (Part 1d) and read the output. To wipe ETOR’s own profile
(settings/cache), delete
`prefix/drive_c/users/<you>/AppData/Roaming/etor/` and relaunch.

**Season compatibility.** ETOR v1.2.0 is the “Afterlight” build (season SS13). It
makes no claim about other seasons, and because parsing happens remotely, support
is whatever the remote site currently implements. The original Chinese app is the
authoritative source.

---

## Appendix A — for AI agents (deterministic replication)

Goal: extract, create a prefix, make text render, launch, and verify — all
verifiable. Every step is idempotent.

**0. Discover inputs**

```bash
wine --version
python3 -c 'import fontTools; print(fontTools.version)'
fc-match -f '%{file}\n' 'Noto Sans'          # -> NotoSans-Regular.ttf
fc-match -f '%{file}\n' 'Noto Sans CJK SC'   # -> NotoSansCJK-Regular.ttc
find "$HOME" -maxdepth 6 -type d -name drive_c 2>/dev/null   # locate prefixes
```

**1. Extract (use `unzip`, never `7z`)**

```bash
mkdir -p ~/Applications/ETOR && cd ~/Applications/ETOR
unzip ~/Downloads/TLI-tracker-translated-v1.2.0.zip -d app
test -d app/locales && test -n "$(find app -maxdepth 1 -iname '*-ETor.exe')"
```

The archive stores Windows backslashes (`locales\af.pak`). `unzip` normalises
them; `7z` and a naive `python -m zipfile` do not. With the latter, normalise
first: `m.filename = m.filename.replace('\\', '/')` before `extract`.

**2. Create the prefix**

```bash
export WINEPREFIX="$HOME/Applications/ETOR/prefix" WINEARCH=win64 WINEDEBUG=-all
wineboot -u
wine reg query 'HKLM\Software\Microsoft\Windows NT\CurrentVersion\Fonts' >/dev/null
```

**3. Create the alias fonts** — identical to the Part 2 script, then:

```bash
fc-cache -f "$HOME/.local/share/fonts/etor-alias"
fc-match 'Segoe UI'          # must print a path under etor-alias
fc-match 'Microsoft YaHei'   # must print a path under etor-alias
wineserver -k 2>/dev/null || true; sleep 1
wineboot -u
wine reg query 'HKLM\Software\Microsoft\Windows NT\CurrentVersion\Fonts' | grep -i 'Segoe UI'
```

Do **not** copy fonts into `$WINEPREFIX/drive_c/windows/Fonts/`; it does not
register them here.

**4. Launch with a debug port**

```bash
cd ~/Applications/ETOR/app
setsid nohup wine ./*-ETor.exe --no-sandbox --disable-gpu \
  --remote-debugging-port=9222 --remote-allow-origins='*' >/tmp/etor.log 2>&1 &
sleep 12
```

**5. Verify over CDP** — get the page target, then `Runtime.evaluate`:

```bash
WS=$(curl -s http://127.0.0.1:9222/json/list \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)[0]["webSocketDebuggerUrl"])')
```

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

| key | broken | fixed |
| --- | --- | --- |
| `sans` | `0` | `~48–58` |
| `segoe` | `0` | `~58` |
| `yahei` | `0` | `~52` |
| `textLen` | `~32` (digits only) | hundreds (labels present) |

If `sans` stays `0`, the Wine font registry did not refresh — re‑run
`wineserver -k` + `wineboot -u` in the correct prefix, then relaunch.

**6. Rollback**

```bash
rm -rf ~/Applications/ETOR
rm -rf ~/.local/share/fonts/etor-alias && fc-cache -f
```

The one step an agent cannot do head‑less is **Part 3b** — the log path is entered
in the GUI and stored in the Electron/IndexedDB profile, though it can also be
driven over CDP if needed.

---

## Appendix B — a possible app‑side fix (upstream)

Adding Linux fallbacks to the font stack would fix Part 2 for everyone, with no
per‑user workaround:

```css
font-family: "Segoe UI", "Microsoft YaHei", "Noto Sans", "Noto Sans CJK SC",
             system-ui, sans-serif;
```

Both `"Noto Sans"` and `system-ui` measure non‑zero under Wine (verified). This CSS
lives in the **remote** app, so only the original authors can change it for good.
However, the translation layer here *is* injected into every page
(`build/obf-app/translate.js`, loaded via `mainWindow.webContents.executeJavaScript`
in `main.js`), so a maintainer could ship the fallback locally by prepending:

```js
const s = document.createElement('style');
s.textContent = 'html,body,button,input,select,textarea{font-family:' +
  '"Segoe UI","Microsoft YaHei","Noto Sans","Noto Sans CJK SC",system-ui,sans-serif !important}';
(document.head || document.documentElement).appendChild(s);
```

Caveat: that means repacking `resources/app.asar`, and the override needs
re‑checking whenever the remote styling changes.

---

## Recommendation

Best raised with the original YiHuo ETor developers — as `README.md` notes, core
feedback goes to them. In the meantime, Parts 1–3 get a Linux user to a working,
text‑readable install, and Part 2 restores the UI for anyone who already has it.
