# Carusela brand assets

The plugin uses the same C mark and forest-green / lime palette as [Carusela](https://carusela.com).

| Asset | Use |
|---|---|
| [Logo mark](logo-mark.svg) | Square icon, with a transparent surround |
| [Wordmark](wordmark.svg) | Compact Carusela lockup on a light background |
| [Banner source](banner.svg) | Editable SVG for the repository header |
| [Social preview](social-preview.png) | 1280 × 640 PNG, also used in the README |

The PNG includes its background and works on both light and dark pages. The SVGs are
self-contained, with no scripts, external fonts or image requests.

## Palette

| Colour | Hex |
|---|---|
| Forest | `#163300` |
| Deep forest | `#0e2200` |
| Lime | `#9fe870` |
| Soft lime | `#d6f7bf` |
| Mist | `#f4f7f1` |

Keep the logo's proportions and the contrast between the mark and its background.
Use **Carusela** in English and **קרוסלה** in Hebrew. The plugin's full name is
**Carusela for Claude Code**; its install identifier is `carusela@carusela`.

## Updating the banner

Edit `banner.svg`, then regenerate the PNG from the repository root with
[librsvg](https://gitlab.gnome.org/GNOME/librsvg)'s renderer:

```sh
rsvg-convert --width 1280 --height 640 assets/brand/banner.svg \
  --output assets/brand/social-preview.png
```

The banner uses Arial / Helvetica with a sans-serif fallback and Menlo / Consolas for
the sample prompt. Check the rendered PNG after editing, including at README width.
If the skill count changes, update it in the banner too.

Repository maintainers can also upload `social-preview.png` under GitHub repository
**Settings → General → Social preview**. Adding the file alone does not set that preview.
