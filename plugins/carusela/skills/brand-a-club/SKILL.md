---
name: brand-a-club
description: Change a Carusela club's colours, logo, favicon, apple icon and social share card. The Carusela MCP publishes a design change in the same call. Use when somebody wants to brand or rebrand their club, change its colours, upload a logo, or fix how it looks when shared.
---

# Brand a club

## What this skill forbids

**Never publish design the owner did not ask for.** `preview_design_change` publishes in the
call itself: the change is live for members when the result comes back. There is no token, no
draft to approve and no second call. The owner asking for the change is the approval, so make the
call in the same turn and send only the fields the request covers. When the request leaves the
values open ("make it look warmer"), choosing them is part of the job; tell the owner which
colours you chose and why.

**Never pass `confirmation_token`.** No tool takes it any more. A tool that receives it refuses
the call and the refusal names the retired argument. Call again without it, after reading the
current state.

**Never put a colour or an asset through `update_club_copy`.** It refuses them and says where
they belong. That refusal is the system working.

## One call

```
get_config             -> what is live, an unpublished admin draft if one exists, home_blocks
preview_design_change  -> { published, version, changes, summary, live_url, admin_url }
```

Read `get_config` first, and keep that read: it is what you send back to undo.

- **A pending admin draft goes live with your change.** If somebody saved an unpublished edit in
  the admin brand or home screen, `get_config` shows it, and `preview_design_change` publishes it
  together with your fields. Both appear in the same `changes`. Tell the owner which lines came
  from the admin draft.
- **Nothing to change answers `already_live`** and writes nothing, so a repeated call is safe.
- **`publish_design` takes no argument.** It publishes a draft somebody saved in the admin brand
  or home screen. Call it only when the owner asks for that admin draft to go live.

There is no look-first mode for design over MCP: whatever `preview_design_change` receives goes
live. An owner who wants to try a look before members see it saves a draft in the admin brand
screen, and `publish_design` publishes it when they ask.

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
| `logo` | header lockup, light backgrounds | wide PNG or vector SVG, transparent |
| `logo_dark` | same lockup for dark mode | same, light ink |
| `favicon` | browser tab | square PNG, 192px |
| `apple_icon` | iOS home screen | square PNG, 180px |
| `og_image` | social share card | **1200x630 PNG** |

Upload each one first, through the two calls that strip the file's metadata, because a photo's
GPS location would otherwise be public:

1. `attach_media` `action: "stage_upload"` with a filename and the field as `image_surface`
   (`logo` for both logos, `favicon`, `apple_icon` or `og_image`)
2. PUT the bytes to `upload_url`. When stage returned `upload_headers`, send exactly those;
   otherwise send the file's `Content-Type` (`image/png`, `image/jpeg`, `image/gif` or
   `image/webp`). A HEIC photo is refused: HTTP 400 with `"statusCode": "415"` (`invalid_mime_type`) in the body
3. `attach_media` `action: "finalize_upload"` with the `staged_path` from step 1, unchanged
4. Use the `public_url` that finalize returns in `preview_design_change`, never before: nothing
   is published until finalize succeeds

PNG, JPEG, GIF, WebP and vector SVG up to 5 MiB all go this way. An SVG needs a filename ending
in `.svg`: stage then returns `upload_headers: {"Content-Type": "text/plain"}`, and a PUT with
`image/svg+xml` instead is refused: HTTP 400 with `"statusCode": "415"` (`invalid_mime_type`) in the body. Finalize parses the SVG and publishes plain vector markup
only; its title, description, metadata and comments are removed. `upload_target` is disabled:
it refuses with "upload_target is disabled" and names these same steps.

Targets are single-use and expire, so stage one per file, close to when you upload it.

`already_published: true` from finalize is a success: an earlier finalize published that file, so
use its `public_url`. A refusal from finalize has already removed the staged copy, except where
it says the read failed, so:

- **larger than 5 MiB**, **not a supported PNG, JPEG, GIF, WebP or vector SVG** (a HEIC photo or
  a PDF, whatever its name says), **its metadata could not be read**: shrink it, convert it, or
  re-export it as a fresh PNG or JPEG, then start again at `stage_upload`. If a re-export fails
  the same way, tell the owner which file and the reason after the colon.
- **vector SVG needs a .svg filename**, in that same refusal: when the file really is an SVG, it
  was staged under another name. Stage it again with a `.svg` filename and PUT it with the
  returned `upload_headers`.
- **the .svg file contains raster bytes**: a PNG or JPEG renamed to `.svg`. Stage it again under
  its real extension.
- **a reason after the colon that starts with `svg:`**: the SVG holds something finalize will
  not publish, such as a `<style>` block (Illustrator classes), Inkscape `sodipodi` markup, an
  embedded `<image>`, a script or event handler, animation, `<foreignObject>`, a DTD, or a
  reference to anything outside the file (only `#id` references pass). Re-export it as plain
  shapes with inline fills, or render it to PNG, then start again at `stage_upload`.
- **nothing staged at**: the PUT never landed or the target expired. Stage and PUT again.
- **not a path stage_upload issued to you**: pass `staged_path` exactly as stage returned it.
- **reading or downloading the staged upload failed**: the staged copy is still there. Call
  `finalize_upload` again with the same `staged_path`.
- **publishing the image to cms-images failed**: tell the owner the reason it gives. Stage again
  only if that reason is temporary.

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
section), and anything in Ring C, which has its own tools (see `build-sales-funnel`).

## After publishing

The result's `changes` and `summary` name every field that moved, from what to what, and for
`home_blocks` which blocks were added, removed or moved. Tell the owner all of it, including what
came from a pending admin draft. `get_config` shows the new `version` and the stored
`brand.assets` and `tokens`. To see it truly live, load `live_url`: the config is one thing and
the rendered page is another, and a stale page is usually just a cache.

The logged-out pages render in light mode regardless of the viewer's system setting, so the dark
palette cannot be checked from there. Say that rather than claiming dark mode is verified.

## Undo

Every publish is a version. Two ways back:

- **Call `preview_design_change` again with the old values.** For colours and assets, the
  `changes` of the call you are undoing hold them. For `home_blocks`, send the whole list that was
  live before the change, because the diff is block by block and the field replaces the list
  whole. Take it from the `get_config` read before the change only when its `home_blocks_source`
  was `published`; when it was `draft`, that list was an unpublished admin draft, and the live one
  is `published.pages.home.blocks` in the same read.
- **Revert a version in the admin.** The history is its own tab, `/admin?tab=brand-versions`
  ("גרסאות"), not the screen `admin_url` opens. Restoring a version there loads that whole
  configuration into a draft that the owner then publishes, so copy and mentor edits published
  since go back with it. Say so before suggesting it.

There is no MCP rollback tool for design; one request back exists only for Sales pages and
Funnels.
