---
name: tune-club-mentor
description: Configure a Carusela club's AI mentor - its name, instructions, vocabulary, starter prompts and its FAQ of curated questions and answers - so it answers from the club's own material. Use when somebody wants to set up or improve the AI assistant in their club, load the questions members keep asking, or says it gives generic or wrong answers.
---

# Tune the club mentor

## What this skill forbids

**Never put a secret in `mentor_instructions`.**

It is published with the club's configuration and it is **public**. No API key, no token, no
password, no internal URL, no member's details. This is not a style rule; the field ships to
anyone who can read the config.

**Never put questions and answers in `mentor_instructions`.** The same reason: it is public, and
it is sent to the model on every answer. Recurring questions and the club's answers go in the
mentor FAQ (`manage_mentor_faq`), which the mentor quotes only when a question matches. Visitors
who are not signed in cannot read it; a signed-in member can get the full text of any entry the
mentor could quote to them, so write every entry as something the club would say to that member.

**Never put a member's or customer's name or details in an FAQ entry.** An entry is the club's
answer, spoken to every member who asks something close to it. Write it as the club's position,
not as a reply to one person.

**Never write instructions or entries that let it invent lessons.** A mentor that confidently
names a lesson which does not exist is worse than one that says it does not know, because the
member goes looking. Make "say you do not know and offer the nearest real thing" an explicit rule
in every mentor you write.

## The four fields

All through `update_club_copy`, all Ring A, all published immediately.

| field | what it does | replaces or merges |
|---|---|---|
| `mentor_name` | what members call it | replaces |
| `mentor_instructions` | its persona and rules, max 8000 chars | **replaces the whole block** |
| `mentor_content_terms` | nouns that mean "our own material" | **replaces the list** |
| `suggested_prompts` | the starter chips | **replaces the list** |

The three that replace do not append. Read the current value with `get_config` first, or you
will silently drop what is there.

The 8000 character cap exists because this is sent to the model on **every answer**. Long
instructions are not more control, they are a bill.

## What actually makes the instructions good

Not tone. Structure. A mentor is useful when it can route a question to the exact place the
answer lives, so the instructions should be mostly a map.

Three blocks, in this order:

**1. Who and what.** One paragraph: what the club is, who teaches, who the members are.

**2. The map.** A line per course or section saying what is in it. This is the block that does
the work. Without it the mentor answers from general knowledge and never cites the club.

**3. How to answer.** Language and register (for Hebrew: spoken Israeli, not translated, with
tool names left in English). Short answer first, then where to find it. Who the member is, so
the level is right. And the two prohibitions: do not invent a lesson, and say when something is
not covered.

Add a **sensitive topics** block when the club teaches anything with a footgun: credentials,
payments, anything that can lose data. Say what the mentor must never advise and which lesson it
must point at first.

The recurring questions are not a block here. They are the FAQ below.

## The FAQ: `manage_mentor_faq`

The questions members actually ask, each with the club's own answer. This is the highest-value
part of a mentor and the one most people skip. Pull the questions from real sources: a Q&A
recording, material the owner hands you, the questions in a community feed. Do not open a
mailbox or member messages yourself. Then **rewrite each one** into a clean question and an
answer the club stands behind. Never paste a raw thread.

How an entry reaches a member:

- The mentor quotes an entry's answer as the club's position when a member asks something close
  to its question. Visitors who are not signed in cannot read the FAQ; a signed-in member can get
  the full text of any entry the mentor could quote to them.
- `min_tier_level` holds an entry back from members below that access level (0 is everyone).
- `content_type` with `content_id` links an entry to a lesson, recording, course, tutorial or
  guide: the mentor quotes it only to members who can open that content, and can point to it.
- `is_active: false` keeps an entry without quoting it.
- `embedded: false` on a new or edited entry means it is found by its words until its embedding
  lands, then by meaning too.

What an entry may say:

- A question up to 300 characters and an answer up to 1500. At most 300 entries per club.
- **No URL, email address or phone number.** The tool refuses them. Link the entry to the club's
  content instead; the club's support address lives in its links, not in an answer.
- No customer names, no details of one person's case, no price or policy the club does not
  actually hold. If the owner is unsure of an answer, leave the entry out.

The flow:

1. `list` first. It returns the entries and a `revision`.
2. `upsert` with `expected_revision` and up to 100 entries. An entry without `id` is added; an
   entry with `id` edits that entry and changes **only the fields it carries**, so correcting an
   answer never resets its access level. Each write returns the next `revision`; pass it to the
   next call when loading more than 100.
3. `remove` with `expected_revision` and the `ids` to delete. There is no undo.

Show the owner the questions and answers before loading them. They are spoken in the club's
voice.

When a write is refused:

- **A stale revision** means someone saved in between, maybe from the admin screen. `list`
  again, check the entries you meant to change, and resend with the new revision.
- **A batch with bad entries** is refused whole, with every bad entry named in one message.
  Fix those and resend with the same revision; nothing was written.
- **A lost answer** (a timeout): `list` and check whether your entries are there before
  resending. A repeat with the old revision is refused if the first call landed, so it cannot
  add the same entries twice, but a repeat with a fresh revision would.
- A `warnings` entry about two entries asking the same question usually means a duplicate:
  remove one.

## `mentor_content_terms`

The nouns that make a question count as being about the club's own material rather than the
world. For a workshop: the words for the sessions, the days, the modules, the tools taught. Get
these right and the mentor searches the club before it answers from training data.

Replace the list, do not guess at adding to it: an empty list restores the platform default.

## `suggested_prompts`

Four to six. Make them the questions a **stuck** member has, not the questions a brochure would
ask. "How do I start?" is worth less than "It says it fixed it and it did not, what now?": the
second proves the mentor knows this specific club.

## What is refused here, and where it lives

`mentor_avatar`, `mentor_route_prefixes` (which areas it may link to), `mentor_internal_hosts`
and `mentor_starter_answers` are all Ring B: the mentor section of the brand editor. The refusal
messages name them. Do not look for another tool.

## Verify

`get_config` and read `mentor` back, and `manage_mentor_faq` with `list` for the FAQ. Then ask
the mentor one of the FAQ questions in different words, and something only this club could
answer. A generic answer means the map block or the FAQ is too thin, not that the model is bad.
