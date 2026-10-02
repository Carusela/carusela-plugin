---
name: brand-a-club
description: Change a Carusela club's colours, logo, favicon, apple icon and social share card through the gated design flow. Use when somebody wants to brand or rebrand their club, change its colours, upload a logo, or fix how it looks when shared.
---

# Brand a club

## What this skill forbids

**Never call `publish_design` without showing the human the preview first.**

`preview_design_change` returns a `preview_url` and a single-use `confirmation_token`. The token
exists so that a person sees the diff before members do. Staging and publishing in one breath
because you are confident spends the token on a review that never happened, and turns a designed
gate into a formality.

Show the summary and the link. Then publish.

**Never put a colour or an asset through `update_club_copy`.** It refuses them and says where
they belong. That refusal is the system working.

## The two steps

```
preview_design_change  -> { changes, preview_url, confirmation_token, expires_at }
publish_design         -> { published: true, version }
```

The token is single-use, short-lived, and bound to the club, the account and the exact draft it
was issued against. If it expires or somebody else saves a draft in between, preview again. Do
not retry a spent token.

## Colours

Four tokens, all **HSL triples with no `hsl()` wrapper and no commas**:

```
color_primary        "15 63% 52%"     light mode
color_accent         "22 48% 90%"     light mode
color_dark_primary   "16 79% 75%"     dark mode
color_dark_accent    "15 22% 24%"     dark mode
```

What they mean in this design system, which is not obvious from the names:

- **`primary` is the ink.** Brand-coloured text, links, active states. In light mode it wants to
  be dark enough to read on white; in dark mode light enough to read on near-black.
- **`accent` is the surface.** Soft fills, the main button's background, tinted panels. In light
  mode it wants to be a pale tint (high lightness); in dark mode a deep one (low lightness).

So the pair inverts between modes: light gets a **dark primary and a pale accent**, dark gets a
**pale primary and a deep accent**. Setting all four to the same saturated colour produces a
club that is unreadable in one of the two modes, and you will not see it, because the logged-out
pages render light only.

Converting a brand hex to HSL: hue and saturation stay, lightness is what you move per mode.

## Assets

Five, all URLs that must come from `attach_media`:

| field | what it is | make it |
|---|---|---|
| `logo` | header lockup, light backgrounds | wide PNG, transparent |
| `logo_dark` | same lockup for dark mode | same, light ink |
| `favicon` | browser tab | square PNG, 192px |
| `apple_icon` | iOS home screen | square PNG, 180px |
| `og_image` | social share card | **1200x630 PNG** |

Upload each one first:

1. `attach_media` `action: "upload_target"` with a filename
2. PUT the bytes to `upload_url` with the correct `Content-Type`
3. Use the returned `public_url` in `preview_design_change`

Targets are single-use and expire, so request one per file, close to when you upload it.

`og_image` is the SEO card, **not the logo**. It carries the club name and the promise, sized
for a link preview. Putting a bare logo there wastes the one image a share shows.

## Making the assets, when the club has none

Compose an SVG and render it. For right-to-left text, two things matter:

- **`text-anchor` and `direction="rtl"` are unreliable in SVG rasterisers.** Render each line
  left-anchored on a wide canvas, trim it, and composite it where you want it.
- **A line mixing Hebrew and Latin needs an explicit RTL embedding** (`&#x202B;` … `&#x202C;`)
  or the Latin run lands on the wrong side. Pure-Hebrew lines are fine without it.

Always look at the rendered PNG before uploading it. A bidi bug is invisible in the markup and
obvious in the image.

## What this flow will not change

Refused here, with a real door elsewhere: tracking pixels and analytics ids (admin settings,
ANALYTICS tab), the navigation rail (admin settings, CHROME tab), feature flags (admin,
"יכולות הקהילה" tab), the mentor's avatar and its allowed link areas (brand editor, mentor
section), and anything in Ring C, which has its own preview-and-approve flow.

## After publishing

`get_config` shows the new `version` and the stored `brand.assets` and `tokens`. To see it
truly live, load the club's own URL — the config is one thing and the rendered page is another,
and a stale page is usually just a cache.

The logged-out pages render in light mode regardless of the viewer's system setting, so the dark
palette cannot be checked from there. Say that rather than claiming dark mode is verified.
