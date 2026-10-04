---
name: seed-club-content
description: Load a club's courses, recordings, tutorials, guides and AI agents through the Carusela MCP without losing fields or publishing things nobody reviewed. Use when somebody wants to import, migrate, bulk-create or restructure club content, or asks to turn an existing site, syllabus, video library or another course platform (Schooler, Rav Messer, Teachable) into a club.
---

# Seed club content

## What this skill forbids

**Never invent content that has no source.**

If the syllabus lists a lesson and there is no recording for it, the lesson does not get a made
up video id, and it does not get quietly dropped either. Create it with the sources that exist
and say in the report which items had no video and why. A club full of dead players is worse
than a club with a gap the owner knows about.

**Never publish a batch before the owner has seen it, in a club that has already launched.**
There the create is a publication: members are in, and a course that appears while you are still
building it has been seen. Create, review, then publish.

Before Launch this does not apply, and following it anyway is the mistake. Nobody can reach a
pre-launch club, so a draft hides the import from the one person who wanted it, and the owner
watching the club fill up sees an empty screen. `manage_course` and `manage_library_item` publish
on create while the club has not launched; let them.

**Never write a second copy because the first one refused a field.** A refused field means that
kind has no column for it. Find where the value belongs; do not create a different content type
to hold it.

**Never put another platform's id where a member can see it.** `tag` on a course, tutorial or
guide is the chip on its card and in its header, and `tags` on a library item are labels and
category joins. Every member reads them. `schooler:45334` in a tag is not bookkeeping, it is a
string printed on every course the club sells. This has already happened on a real club.

The source reference belongs in `import_source` (`"schooler"`) and `import_ref` (`"45334"`).
Check the tool's input schema in `tools/list`: where a tool accepts those two fields, use them;
where it does not yet, **do not store the reference in the club at all** and keep the mapping from
source id to new id in your report to the owner instead. A `tag` is only ever a short word a
member should see, like `חדש`.

## Order of work

1. `list_my_clubs`, `get_config`, `get_club_overview`. Know the club before writing to it.
2. Decide the shape on paper first: which courses, which chapters, what belongs in the library
   instead. Show it to the owner. Restructuring 70 lessons afterwards is many calls.
3. Create categories (`manage_category`) before the content that references them.
4. Create content.
5. Publish and gate deliberately.
6. `audit-club-content`.
7. Generate covers for what still has none. See Covers below.

## Moving a club from another platform

When the owner is leaving Schooler, Rav Messer, Teachable or similar, decide first what is moving:

- **Content only** (courses, lessons, video links): the tools in this skill. Keep every source id
  out of member-visible fields, as above, and report the id mapping.
- **Members and their access too**: read `club-migration://contract` before any write. It is the
  flow built for a whole-club move: a session, a validated manifest, a plan the owner approves once,
  resumable batches. It asks for record hashes and file digests, so it needs a session that can run
  code (Claude Code with a shell). If you cannot compute what it asks for, say so to the owner
  rather than recreating members by hand.
- **Payments never move.** No card numbers, CardCom tokens, Schooler credentials or active
  subscriptions go into Carusela by any path. A member's paid access in the new club comes from a
  fresh checkout; say that to the owner up front.

Before the first write, run `list_content` for the types you are about to create. A club that
already holds half of the import gets duplicates otherwise.

## Courses carry their lessons

`manage_course` takes `lessons` in the same call. Do not create a course and then add lessons
one by one; it is slower and the ordering is harder to get right.

- `chapter` on a lesson groups it. Lessons sharing a chapter name **consecutively** render as
  that chapter, so order matters more than the string does.
- Lessons are created in array order. `lesson_order` overrides it and **must list every lesson
  id in the course** — a partial list is refused.
- On update, a lesson entry with an `id` updates; one without an `id` is appended.

Per lesson, the fields worth filling: `title`, `description`, `chapter`, `video_url`,
`resource_links`, `duration_label`, `min_tier_level`.

## `resource_links` is not JSON

Newline-separated `Label|https://url` entries. Not an array, not JSON.

```
מדריך ההתקנה|https://example.com/guides/install.html
code.visualstudio.com|https://code.visualstudio.com/
```

It exists on lessons, tutorials, recordings and AI agents. **Guides do not have it** — a guide's
links go inside its `sections` text.

## Video: let the provider fill it in

Pass a Bunny, Vimeo or YouTube URL as `video_url` and leave `thumbnail_url` empty. Lessons,
recordings, tutorials and AI agents all resolve the poster frame from the provider on write, and
lessons and recordings resolve the runtime too. A value you pass is never overwritten by that, so
only pass one if you mean to override.

To upload an image rather than reference a video, use the two calls that strip its metadata,
because a photo's GPS location would otherwise be public: `attach_media` with
`action: "stage_upload"` and a filename, then PUT the bytes to the `upload_url` it returns with
the file's `Content-Type`, then `attach_media` with `action: "finalize_upload"` and the
`staged_path` stage returned, unchanged. Put the `public_url` finalize returns on the row, and
only then: nothing is published until finalize succeeds. The target is single-use and expires.

This takes PNG, JPEG, GIF and WebP up to 5 MiB; the PUT answers 415 for an SVG or a HEIC photo.
An SVG cannot be staged, so only an SVG uses `action: "upload_target"`, the PUT and its
`public_url`. `upload_target` stores the file as sent, metadata included: never use it for
anything else, and never to get past a refusal.

`already_published: true` from finalize is a success: an earlier finalize published that file, so
use its `public_url`. A refusal from finalize has already removed the staged copy, except where
it says the read failed:

- **larger than 5 MiB**, **not a PNG, JPEG, GIF or WebP image** (a HEIC photo or a PDF, whatever
  its name says), **metadata could not be read**: shrink it, convert it, or re-export it as a
  fresh PNG or JPEG, then start again at `stage_upload`. If a re-export fails the same way, put
  the file and the reason after the colon in the report.
- **nothing staged at**: the PUT never landed or the target expired. Stage and PUT again.
- **not a path stage_upload issued to you**: pass `staged_path` exactly as stage returned it.
- **reading or downloading the staged upload failed**: the staged copy is still there. Call
  `finalize_upload` again with the same `staged_path`.
- **publishing the image to cms-images failed**: report the reason it gives. Stage again only if
  that reason is temporary.

## The four library kinds do not take the same fields

One tool, one shape, four different field sets. An inapplicable field is refused by name rather
than dropped, so a refusal is information, not a failure.

| | recording | tutorial | guide | ai_agent |
|---|---|---|---|---|
| `video_url` | yes | yes | no | yes |
| `resource_links` | yes | yes | **no** | yes |
| `tags` | yes | yes | yes | **no** |
| `category` | **no** | **no** | **no** | **required on create** |
| body | `learning_points` | `steps` | `sections` | — |
| default `is_published`, launched club | draft | draft | **live** | **live** |
| default `is_published`, before Launch | **live** | **live** | **live** | **live** |

`steps` and `sections` are **update only**. Create the item, then update it with its body. That
is two calls per guide and there is no way around it today.

**The `is_published` row is a DEFAULT, not a rule.** All four accept the field on create, so a
batch nobody has reviewed is staged by passing `is_published: false` explicitly. Omit it and
before Launch all four go live; after Launch you get the per-kind row above, where a guide and an
AI agent go live the moment they are created and a recording and a tutorial do not.

The same split applies to courses. `manage_course` publishes a created course, and the lessons
created with it, while the club has not launched, and creates a draft afterwards. Its result says
which happened: `published: true|false` and `publish_default: "pre_launch" | "launched" |
"explicit"`. Read it rather than assuming, and put it in the report: "12 courses, published,
because the club has not launched yet" is the sentence the owner needs, and it is also the one
that tells them Launch will not publish anything twice.

## Categories: the trap

Only `courses`, `groups` and `ai_agents` have a real `category` field.

**Recordings, tutorials and guides join a category by carrying its exact name in `tags`.** So:

```
manage_category  scope: "recording"  name: "לייבים מלאים"
manage_library_item  kind: "recording"  tags: ["לייבים מלאים", "יום 1"]
```

The name in `tags` has to match the category name **character for character**. `"לייב מלא"`
against a `"לייבים מלאים"` category attaches to nothing, and nothing tells you. After creating
categories, check they are not empty — `audit-club-content` does this.

`scope: "recording"` is the shared pool for recordings, tutorials, guides and groups.
`scope: "course"` is the separate axis courses use. There is no third pool.

## `difficulty` is a closed set

Exactly four values, on recordings, tutorials and guides:

```
מתחילים · ביניים · מתקדמים · כל הרמות
```

Anything else fails with a raw Postgres constraint name. `"בינוני"` and `"מתקדם"` are both
wrong and both look right.

## Publishing and ordering

`set_content_visibility` handles `is_published`, `min_tier_level`, `is_featured` (recordings
only) and `order`.

`order` is **whole-collection**: it must contain every id in that collection, in the order you
want. A partial list is refused. One call with the full list sets the order; you do not order
items one at a time.

Groups have no published state at all. `is_private` is their only visibility axis.

## Covers

Once content is created and reviewed, whatever still has no cover image is the last thing to
close. Two tools do this: `generate_cover` makes one item's image, `generate_missing_covers`
walks the whole club.

Check the tool exists first. If `generate_missing_covers` is not in `tools/list`, the club is on
an older MCP version that does not have it. Say so and skip this step entirely; do not try to
fake it with `generate_cover` in a loop.

Generation runs on the club's AI key, which is Carusela's unless the owner has set their own in
admin > integrations. Either way it is counted against the club's daily cover limit.

Start with a dry run. Its `club_launched` field decides everything below:

```
generate_missing_covers  dry_run: true
```

**Before Launch, do not ask.** A club with no covers is a club of grey cards, and there is nobody
to show them to yet. Finish every pre-launch batch of content by closing the cover backlog:

```
generate_missing_covers  limit: 5
```

Five is the default pre-launch, because a cover takes ten to twenty seconds and a batch has to
come back. Keep calling it until `remaining_missing` is 0 or it stops on a refusal, and report
each batch as it lands (generated, skipped, how many are still missing) with the club's URL, so
the owner watches the cards fill in instead of waiting for a final count.

**After Launch, ask once.** There the covers are appearing in front of members and the spend is a
decision somebody should make on purpose. Tell the owner how many items have no cover and what it
costs, ask, and generate nothing until they say go. Then the same loop, at whatever `limit` they
agreed.

A batch can end on `stopped_reason` before it runs out of items:

- **`cover_generation_disabled`**: the club switched this off. Point the owner at
  admin > integrations > cover generation and stop; nothing else here can turn it back on.
- **`openai_key_missing`**: the club has no AI key it can use. Carusela's platform key normally
  covers this, so seeing it means the club is set to use its own and has not got one on file. The
  owner either clears that setting or adds their key in admin > integrations. There is nowhere in
  MCP to hold a key, so this is not something to retry.
- **`cover_daily_limit_reached`**: the club's daily cap for the day is spent. Tell the owner how
  many items are still missing a cover, and that they can either come back tomorrow or raise the
  limit in the same admin > integrations screen.
- **`presenter_image_not_in_bucket`** and **`presenter_image_unreadable`**: the club has a
  presenter photo set, and it is not a file this club can read: a URL from somewhere else, or an
  upload that did not land. Every remaining cover would fail the same way, which is why the run
  stops rather than producing a backlog of images without the person who is supposed to be in
  them. The owner re-uploads the photo in the club's first-run step or in admin > integrations;
  do not clear the setting to get past it.

Every one of these is the owner's call, not a bug to route around. Report the stop, name the
fix, and move on to closing out the rest of the report.

## Report honestly

At the end, say how many of each type, how many published versus draft **and why**. Before
Launch the answer is "published, because the club is not live yet", and an owner who is not told
that will assume somebody has already seen it. Say what was gated to which tier, and **what had
no source**. If a category came out empty, say so rather than letting the owner find it.
