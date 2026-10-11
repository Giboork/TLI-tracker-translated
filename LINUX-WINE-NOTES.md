# Running YiHuo ETor on Linux / Steam Deck (Proton & Wine)

> **Status:** exploratory community note — **not a fix**, and **not** from the
> original developers. It documents a rendering problem seen when the Windows
> build is run under Wine, plus two ways to work around it. Offered only because
> Linux/Steam Deck users hit a complete (but easily worked-around) breakage.
> Feel free to close, move, or reword this.

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
c.font = '16px sans-serif';   c.measureText('MMMM').width;  // 0   ← the bug
c.font = '16px "Segoe UI"';   c.measureText('MMMM').width;  // 0
c.font = '16px "Noto Sans"';  c.measureText('MMMM').width;  // >0  ← works
c.font = '16px system-ui';    c.measureText('MMMM').width;  // >0  ← works
```

## Workaround (user side — no app changes)

Teach Wine that `"Segoe UI"` / `"Microsoft YaHei"` exist by installing fonts that
carry those exact family names (aliases of Noto Sans / Noto Sans CJK), then refresh
Wine's font database:

1. Create the alias fonts with Python + [`fontTools`](https://github.com/fonttools/fonttools):
   clone `NotoSans-Regular.ttf` (family → `"Segoe UI"`) and the SC face of
   `NotoSansCJK-Regular.ttc` (family → `"Microsoft YaHei"`) into
   `~/.local/share/fonts/`.
2. `fc-cache -f`
3. `WINEPREFIX=<etor-prefix> wineboot -u` (once) so Wine re-registers host fonts.
4. Relaunch the app.

This is entirely user-side, touches neither the app nor its files, and is
reversible (remove the fonts and re-run `fc-cache`). It also repairs the generic
`sans-serif` mapping for other Wine apps in that prefix.

## Possible app-side alternative (only actionable by whoever controls the UI)

Adding Linux-available fallbacks to the font stack would fix it with no OS changes:

```css
font-family: "Segoe UI", "Microsoft YaHei", "Noto Sans", "Noto Sans CJK SC",
             system-ui, sans-serif;
```

Both `"Noto Sans"` and `system-ui` measure non-zero under Wine (verified). However
this CSS lives in the **remote** app, so the translation layer here cannot change
it without repacking `resources/app.asar`, which would be overwritten on each
release and is not something this repo should carry.

## Recommendation

Best raised with the original YiHuo ETor developers — as `README.md` already notes,
core-functionality feedback goes to them. In the meantime, the font-alias
workaround above restores the UI for affected users.
