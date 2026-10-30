---
name: gate-club-access
description: Decide which Carusela content each access tier can reach and when it opens, and apply it. Use when somebody wants to gate content, build a free tier, put material behind a paid level, open a course or lessons on a date or day by day from each member's join date, or asks why a member cannot see something.
---

# Gate club access

## What this skill forbids

**Never invent a tier level.** `min_tier_level` must be a level the club's tiers table already
defines. A club with levels 0, 1 and 2 will refuse 3, and it is right to. Read the ladder before
you write to it.

**Never gate content you have not looked at.** Moving a lesson to level 2 takes it away from
everyone below. That is a revenue decision wearing the clothes of a config change, and the owner
makes it, not you. When the owner named the rung for the content, apply it. When they asked for a
ladder without saying which content sits on which rung, ask them which rung; that is the decision
itself, not a confirmation step.

**Never claim you set up a subscription.** You did not. Gating content by tier is not the same
as creating a product, a price or a plan. The second thing is Ring C, and it goes through its own
stage-then-apply tools, described below.

## Read the ladder first

`get_club_overview` returns it as `access_tiers`, and you are calling that before any write
anyway:

```json
"access_tiers": [
  { "level": 0, "name": "חינם" },
  { "level": 1, "name": "בסיסי" },
  { "level": 2, "name": "פרימיום" }
]
```

`get_member_stats` with `include_tiers: true` also returns it, with a member count per tier.
Use that one only when the counts are the question: it is a tool about people, and gating content
is not.

## Apply it

`set_content_visibility`, one call per item, works across every content type:

```
kind: course | lesson | event | group | recording | tutorial | guide | ai_agent
id: <the item>
min_tier_level: <a level the club defines>
```

The same tool carries `is_published`, `is_featured` (recordings only) and `order`. An axis a
type does not have is refused by name rather than ignored, so a change that cannot happen never
reports success.

**A course and its lessons are gated separately.** Setting the course does not cascade. If the
owner wants a free preview lesson inside a paid course, that is exactly how you build it: course
at the paid level, one lesson at 0.

## A ladder that works

The shape most clubs want, and the reasoning:

- **Level 0 is the proof.** Whatever makes a stranger believe the club is worth paying for.
  Setup and prerequisites are good here: they cost the owner nothing to give away, and someone
  who has installed the tools has already invested.
- **Level 1 is the product.** The thing they paid for.
- **Level 2 is the depth.** Bonuses, archives, Q&A, the agents and automations.

Put the prerequisites free even when the course is paid. A member who cannot complete the setup
cannot use what they bought, and support absorbs the difference.

## Open content on a schedule

Tiers decide **who** gets in. The release schedule decides **when**, for people who already have
access. It never opens anything to someone without access, and it never takes away what they own.
Read the schedule first:

```
get_course_schedule  course_id: <course>  [preview_day: N | preview_at: <ISO time>]
```

The answer has four parts:

- **The course settings.**
  - `schedule_mode` is `dates` (fixed dates, the same for everyone) or `member_start` (each member
    counts days from the day they bought or joined).
  - `release_hour` is the hour that day 2 onward opens, in Israel time.
  - Also `release_at`, `pre_release_visibility`, `default_item_visibility` and `notify_on_release`.
- **Every section and lesson.** Each has its own rule and its effective rule. A lesson never opens
  before its section or its course. `overridden: true` means its own earlier rule is ignored.
- **The preview.** `released` at the moment you previewed.
- **Warnings.** Offers whose access ends before the last lesson opens.

How to set it:

| The owner wants | Call |
|---|---|
| The whole course opens on a date (presale) | `manage_course` update, `schedule: {course: {release_at: "2026-11-05T08:00:00+02:00", pre_release_visibility: "teaser"}}` |
| A drip by days since each member joined | `manage_course` update, `schedule: {course: {schedule_mode: "member_start"}, sections: [{id, release_offset_days: 0}, {id, release_offset_days: 7}, ...]}` |
| One lesson or the course on its own | `set_content_visibility` with `release_at`, `release_offset_days` (lessons) or `pre_release_visibility` |
| New sections with their dates | `save_course_structure` with `release` |

**Getting the values right:**
- **Day N is `release_offset_days` N - 1.** Day 1 opens at the member's start. A weekly course
  opens week N on offset 7(N - 1).
- **Times carry their offset.** Never send a time with no offset. 02:00 to 02:59 Israel time is
  refused, because that hour does not exist on the night the clocks move forward.
- **`teaser` shows the item with its date as coming soon. `hidden` leaves it out until it opens.**
  Recordings, tutorials, guides, events and groups have no schedule, and the tools refuse one by
  name.

**When a change would close lessons members already opened, nothing is written.** The answer is
`status: confirmation_required` with `relocked_members` and `relocked_lessons`. Moving a lesson
into a later section in `save_course_structure` counts too. Tell the owner the numbers in plain
words. Only if they agree, send the same call again with `confirm_relocked_members` set to that
member count. Never confirm on your own.

## When somebody asks for a price, an offer or a new tier

"Set up a subscription", "make this cost 200 shekels", "add a coupon", "start a trial": these are
Ring C. Stage with `manage_membership_tier`, `manage_offer` or `manage_coupon`, then call
`apply_commerce_changes` with the draft ids and revisions in the same turn: the owner's request is
the approval, and the apply happens in that call. If the request leaves the price or the tier's
level open, ask for it first. A new tier is not a rung of the ladder until it is applied, so gate
content against it afterwards, not before. Check `get_commerce_readiness` before promising a paid
offer: without a connected terminal it can be staged and cannot be sold. See `build-sales-funnel`
for the fields.

## Verify

`list_content` with `min_tier_level` returns only items requiring exactly that level. Three calls
give you the whole ladder and are the fastest way to show an owner what each tier actually
reaches.
