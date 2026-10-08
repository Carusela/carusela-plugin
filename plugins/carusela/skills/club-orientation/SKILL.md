---
name: club-orientation
description: Read this before the first write to a Carusela club. Explains what the MCP can and cannot do, why a refusal is usually correct, and how to pick the right club. Use whenever a Carusela tool is about to be called for the first time in a session, when a tool refuses a field, or when somebody asks what Claude can change in their club.
---

# Carusela orientation

## What this skill forbids

**Never treat a refusal as a bug to route around.**

Every field the Carusela MCP refuses is refused on purpose, and the refusal message names where
that thing actually lives. `update_club_copy` will not set a colour, a price or a feature flag.
That is not a missing feature you should work around by finding another tool. It is the surface
telling you the change belongs somewhere else, usually because it decides money or access.

If a refusal blocks the user's goal, the answer is to say so and name the right door. Never a
workaround.

**Never write before you have read the club's config.** `get_config` is one call and it tells
you the brand, the tiers, the enabled features, the navigation and the mentor. Half the mistakes
in this surface come from writing into a club whose shape you assumed.

**Never guess `club_id`.** If the account reaches more than one club, every tool requires it and
guessing puts a lesson in a stranger's club. Call `list_my_clubs`, and if more than one comes
back and the user has not said which, ask.

**Never add a confirmation step the server does not have.** The design, commerce, Sales page,
Funnel and A/B test tools apply the change in the call itself. There is no token, no draft to
approve and no second call: the owner asking for the change is the approval. So:

- **The request names the change** (a colour, a price, "publish the page"): make the call in the
  same turn. Do not stop to ask "should I publish?".
- **The request leaves a value open** (no price given, no winner named, which lessons go behind
  which tier): ask for the value. That is a question about what to do, not a confirmation.
- **The owner asks to see it first**: the read-only previews for Sales pages and Funnels show it
  without writing (see `build-sales-funnel`). Design has no look-first mode over MCP.

Never send `confirmation_token`. No tool accepts it now; a tool that receives it refuses the call
and the refusal names the retired argument. Call again without it, after reading the current
state.

Exactly two writes still wait for the owner: emailing real people needs their yes to the
recipient count (`confirm_recipient_count`), and a migration import runs only after the owner
approves its plan in the admin screen. An attestation a tool says only the owner can give (the
mailing-list consent on `import_mailing_contacts`) is never supplied on their behalf: ask for it.

## The one thing to do first

```
list_my_clubs            -> which clubs, and the id for every later call
get_config               -> brand, tokens, features, navigation, mentor
get_club_overview        -> how much of each content type exists, published vs draft
```

Three read-only calls. They cost nothing and they answer most of what you were about to guess.

## The tool map, by job

| The user wants to | Tools |
|---|---|
| See what exists | `get_club_overview`, `list_content`, `get_config`, `get_member_stats` |
| Add or edit a course | `manage_course` (lessons come with it, in the same call) |
| Add a recording, tutorial, guide or AI agent | `manage_library_item` |
| Add an event or a group | `manage_event`, `manage_group` |
| Organise | `manage_category`, `set_content_visibility` |
| Change words | `update_club_copy` |
| Change colours, logos or the home page's block order | `preview_design_change`, which publishes in the same call |
| Bring in a video or upload a file | `attach_media` |
| Bring in members from another system | `import_members` |
| See what is for sale, and whether the club can sell | `get_commerce_catalog`, `get_commerce_changes`, `get_commerce_readiness` |
| Set a price, an offer, a trial, a coupon or a member tier | `manage_offer`, `manage_coupon`, `manage_membership_tier`, then `apply_commerce_changes` |
| Build a Sales page and its Funnel | see `build-sales-funnel` |
| See what has been done | `get_audit_log` |

## The three rings, in one paragraph each

**Ring A is words.** Club name, tagline, page headings, the mentor's text, copy fragments,
outbound links. `update_club_copy` writes them and the platform publishes them on write, with no
second step. That is how the platform is built, and it is not a reason to skip asking: if the
user has not told you what the new wording should be, get it from them before you write it. What
the ring being ungated means is that a wrong word is cheap to correct, not that nobody needs to
see it.

**Ring B is design.** Colours, brand assets and the home page's block composition.
`preview_design_change` publishes them in the call itself and returns exactly what changed, from
what to what. Despite its name it is not a preview. A design draft somebody saved in the admin and
did not publish goes live with it and shows in the same diff; `get_config` shows whether one
exists. `publish_design` takes no argument and publishes only that admin draft, when the owner
asks. The home page's `home_blocks` is a whole-list replace: read the current list with
`get_config` and send all of it, because a block you leave out is removed. To undo, call
`preview_design_change` again with the old values, or revert a version in the admin versions tab
(`/admin?tab=brand-versions`).
See `brand-a-club`.

**Ring C is money and access.** Prices, offers, trials, instalments, coupons and member tiers.
Only an owner or an admin of the club can call these tools. They stage, then apply, both in the
same turn when the owner asked for the change:

1. **Stage.** `manage_offer`, `manage_membership_tier` and `manage_coupon` each take the
   complete desired values, never a patch, plus a `request_id` you reuse on a retry. Staging
   changes nothing a member can see: a new resource stays unpublished or inactive, and an
   existing live one is untouched until apply. To revise a staged draft, pass its `draft_id`
   with its exact `expected_revision`; `get_commerce_changes` lists your open drafts. If the owner
   decides against a staged change, `abandon_commerce_change` takes it back.
2. **Apply.** `apply_commerce_changes` takes the draft ids and revisions (and, optionally, Sales
   page ids and draft course ids to publish in the same batch) and applies them in that call. It
   returns what was applied and the buyer preview. The batch applies whole or not at all: a draft
   no longer staged at that revision, or a live row that changed since staging, refuses all of
   it. `preview_commerce_changes` writes nothing and only points at `apply_commerce_changes`;
   skip it. To undo, stage the previous values and apply them the same way.

What the server checks before it stages anything: an offer sells exactly one product (a member
tier or a course), its price is a whole number of agorot, its interval is `one_time`, `monthly`
or `yearly`, a one-time offer has no trial days, instalments are allowed only on a one-time offer
and then need at least 2. A percent coupon cannot exceed 100; a fixed one is in agorot. A member
tier here is the club's own membership level, not the platform package the club pays Carusela
for.

**What stays human-only in Ring C: connecting the club's CardCom terminal.** A person does it by
hand in the admin payments tab, `/admin?tab=payments`, and the terminal credentials never pass
through MCP. `get_commerce_readiness` says where the club stands: `ready`, `not_connected`,
`not_synchronized` or `unknown`. Drafts can be prepared in every state; `can_publish_paid_offers`
requires both `status: "ready"` and `support_email_set: true`. Missing terminal readiness belongs
in the payments tab. A missing support email needs an address the owner names or confirms:
`update_club_copy { links: { supportEmail } }` or `/admin?tab=contact`. Never publish the sign-in
email as a support address without confirmation. An unknown support-email check gets a retry,
not permission to sell. Nothing in Ring C charges a member or refunds one.

Sales pages and Funnels sit on top of Ring C. Each publishes in one call and rolls back in one
call, or a Sales page and its saved Funnel draft can go live inside the same commercial batch. See
`build-sales-funnel`.

**Feature flags are the owner's, in the admin.** Whether the community, groups, events, the AI
mentor or the agents page is on is switched in the admin under "יכולות הקהילה", `/admin?tab=features`.
MCP reads them through `get_config` and refuses to create content behind a flag that is off; it
does not flip one. The door is that tab, not "ask Carusela".

## What MCP will never do, so stop looking

- Charge a member, refund one, or connect or change the payment terminal
- Email real people without the owner's yes to the recipient count, or run a migration import
  the owner has not approved in the admin
- Switch a feature flag
- Reorder the navigation rail
- Touch a repository, a deployment, DNS or a domain
- Read a member's contact details outside the member tools that hold the `member_pii` capability

Each has a real door: the admin payments tab for the terminal, the owner's yes to the recipient
count for an email, the admin screen for a migration plan, the features tab for flags, the admin
CHROME tab for the rail. Tell the user which door, and say plainly that this surface will not go
around it. Then carry on with the part you can do. A refusal the user cannot act on is worse than
no answer, so the door is the part that matters.

## Three traps that cost real time

**Features can be off, and the surface will tell you.** A club with `features.groups: false` has
no groups area, and `manage_group` now REFUSES to create one rather than leaving an invisible row
behind. Read that refusal as information: the owner switches the flag on at `/admin?tab=features`,
and the refusal names the switch. Ask them to flip it, then create the content. `get_config` shows
which features are on before you plan around one.

**Before Launch, a created course or library item is published, not a draft.** Nobody can reach a
pre-launch club, so the draft would hide the work from the owner rather than protect a member;
after Launch the old per-kind defaults return. `manage_course` says which happened in
`publish_default`. See `seed-club-content`.

**Categories for recordings, tutorials and guides are not a `category` field.** Those three
join a category by carrying its exact name in `tags`. `manage_library_item` refuses `category`
for them, correctly, because the column does not exist. See `seed-club-content`.

## Everything is logged

Every call lands in `get_audit_log` with `source: "mcp"`, including the ones that failed and the
arguments they carried. Use it at the start of a session to see what a previous session already
did, rather than repeating it. Emails in the arguments are redacted; that is deliberate.

## When you are done

Do not report success from the tool result alone. `get_club_overview` after a batch tells you
how many items are published versus draft, and that is the number the user actually cares about.
`audit-club-content` is the fuller version of that check.
