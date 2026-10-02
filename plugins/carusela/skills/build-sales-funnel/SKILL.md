---
name: build-sales-funnel
description: Build and publish a Carusela Sales page and its Funnel (Order bumps, Upsells, Downsells and the Thank You page) through the Carusela MCP. Each price, publication, rollback and A/B test goes live in the call the owner asks for, with no second confirmation call. Use when somebody wants to sell a course or a membership, set up a sales page (from blocks, as their own designed HTML page, or on their own site), a checkout, an order bump, an upsell or downsell, a thank-you page (designed HTML pages for upsells and the thank-you included), a coupon for a launch, an A/B test between funnel versions, or to roll a sales page or funnel back.
---

# Build a sales funnel

## What this skill forbids

**Never put live what the owner did not ask for.** These tools change what a buyer meets, and
each one applies in the call itself: `apply_commerce_changes`, `publish_sales_page`,
`rollback_sales_page`, `publish_sales_funnel`, `rollback_sales_funnel`,
`start_sales_funnel_experiment` and `end_sales_funnel_experiment`. There is no token, no draft to
approve and no second call. The owner asking for the change is the approval, so when they asked
for the page to go live, publish it in the same turn; do not stop to ask "should I publish?". When
the request leaves a price, a coupon or a test's winner open, ask for that value: a price you
invented is a price a buyer pays.

**Never send `confirmation_token`.** No tool accepts it now. A tool that receives it refuses the
call, names the retired argument and says to call again without it after reading the current
state.

**Never use `preview_commerce_changes` or `confirm_sales_funnel_experiment` as a step.** Both write
nothing and answer only with the call to make next. Call `apply_commerce_changes`,
`start_sales_funnel_experiment` or `end_sales_funnel_experiment` directly.

**Never stage or publish again because a repeat was refused.** A call that arrives twice meets
what the first one already did: `already_live`, "nothing staged to publish", a draft no longer
staged at that revision, a test that is already running. Read `get_audit_log` and the current
state; act again only when the owner wants a further change.

**Never promise a buyer can pay before `get_commerce_readiness` says `ready`.** The club's
CardCom terminal is connected by a person, by hand, at `/admin?tab=payments`. Never ask for
terminal credentials and never send them through MCP. Everything in this skill can be drafted
while the terminal is missing; nobody can pay until it is connected.

**Never mix the two page kinds, and never author topology.** A Sales page is either built from
the blocks `get_landing_page_catalog` returns (no colours, CSS, class names, raw HTML, scripts or
iframes inside a block; the page follows the club's published brand on every render), or it is
one designed HTML page the owner wants, uploaded whole and shown in a sandbox (see "A designed
page"). A designed page is the draft's only block. A Funnel is a list of Order bumps, an ordered
list of steps and a Thank You; the server derives the graph, the node ids and whether each step
is an Upsell or a Downsell.

**Never send half a document.** Both `edit_sales_page_draft` and `edit_sales_funnel_draft`
replace the whole draft. A step whose `content` you leave out loses its authored copy, and an
empty `steps` list removes every step. Read first, change what you mean to change, send all of it.

## Who can do this

The owner or an admin of the club. Almost every tool here needs the `commerce` or
`design_publish` capability, and only those two roles hold them. A content editor can read a Sales
page, validate a draft and save one (`get_sales_page`, `validate_sales_page_draft`,
`edit_sales_page_draft`), and nothing else here. When a tool answers that the account's role does
not carry a capability, that is the answer: a person with that seat does it.

## The first calls

```
list_my_clubs             -> which club, and the id for every later call
get_commerce_readiness    -> can this club take money yet, and if not, why
get_commerce_catalog      -> courses, member tiers, offers and coupons, drafts included
list_sales_pages          -> what already exists, and every page id
get_landing_page_catalog  -> the closed block vocabulary a page is built from
```

All read-only. `get_commerce_catalog` pages with `offset` and `limit` (up to 100).

## The flow

Hebrew is the club's language. Every word a buyer reads (offer names, headings, button labels,
the Thank You) is written in Hebrew unless the owner says otherwise.

### 1. Readiness

`get_commerce_readiness` returns `status`: `ready`, `not_connected` (connect CardCom in the
payments tab), `not_synchronized` (connected, but the club's selling state is out of step; check
the existing connection in the admin) or `unknown` (the check could not be completed; retry it).
`can_prepare_drafts` is always true. `can_publish_paid_offers` is true only when `ready`.

Publishing a page is not the same as the page taking buyers. `get_sales_page` returns `readiness`
with two parts. `publish` is what a publish needs: a valid, non-empty page document and a primary
offer that exists, is published and fits the page. `activate` is what a buyer reaching it needs
on top of that: the page published, the club's CardCom terminal connected, a Funnel with a live
version (a Thank You at least) and the club launched. Tell the owner which of these are still open.

### 2. The product

A Sales page sells one product: a course or a member tier.

- A course comes from `manage_course`. See `seed-club-content`.
- A member tier is staged with `manage_membership_tier`: `name`, `description`, `rank` (its
  access level) and `isActive`. It is the club's own membership level, not the Carusela package
  the club pays for.

A page that sells a member tier needs an offer for that tier.

### 3. Offers and coupons

Stage each with its own tool. Every call carries a new `request_id` (a UUID you reuse only to
retry the same request), and `values` holds the complete desired state, never a patch. Omit
`resource_id` to create; pass it to change an existing one. To revise a staged draft, pass its
`draft_id` together with its exact `expected_revision`; `get_commerce_changes` lists your drafts.

`manage_offer` values:

| field | rule |
|---|---|
| `name` | 2 to 120 characters |
| `description` | up to 2000, or null |
| `courseId` / `membershipTierId` | exactly one of the two |
| `priceAgorot` | whole agorot, 0 or more (₪490 is `49000`) |
| `interval` | `one_time`, `monthly` or `yearly` |
| `trialDays` | 0 or more, and 0 on a one-time offer |
| `allowsInstalments`, `maxInstalments` | one-time offers only; `maxInstalments` at least 2 and only with instalments on |
| `isPublished` | the state once applied, default true |
| `orderBumpOfferId` | the offer's own single order bump, or null |

`manage_coupon` values: `code` (stored in upper case), `discountType` `percent` or `fixed`,
`discountValue` (a percent up to 100, or fixed agorot), `maxUses` or null, `validFrom` and
`validUntil` as ISO date-times with an offset (`validUntil` may be null), and `isActive`.

Staging changes nothing live: a new offer, tier or coupon stays unpublished or inactive, and an
existing live one is untouched until apply.

Then apply, in the same turn when the owner asked for the change:

```
apply_commerce_changes  { changes: [{id, revision}], page_ids?, course_ids? }
  -> what was applied, the buyer preview, readiness
```

Up to 50 drafts, 20 pages and 50 courses. It applies them in that call. The batch applies whole
or not at all: a draft that is no longer staged at that revision, or a live row that changed since
staging, refuses all of it and nothing is applied. It never charges anybody.

Tell the owner what went live: every price, interval, trial, instalment count, coupon and access
level, and the price-change effects the result lists for an existing offer (new buyers pay the new
price at once, active subscribers keep the old price until their next renewal, past orders do not
change).

To undo, stage the previous values with the same `manage_` tool and apply them the same way. A
staged change the owner decides against is taken back with `abandon_commerce_change`
(`draft_id`, `expected_revision`).

`page_ids` and `course_ids` make the same batch publish Sales pages and draft courses too. For a
selected page, apply publishes its current draft and, when the page's Funnel has a saved draft,
publishes that Funnel as well, all in one transaction. That is the shortest path for a new club:
stage the offers, build the page and the Funnel on draft offers, then apply the whole launch in
one call. A page in the batch still has to pass its own publish readiness, judged against the
proposed offers.

### 4. The Sales page

```
create_sales_page  { request_id, product_kind, product_id, offer_id, title, slug,
                     entry_mode?, external_url?, seed_fields?, lesson_ids? }
```

- `product_kind` is `course` or `membership_tier`, and `offer_id` is an existing offer for that
  product. Draft offers and draft products are fine at this stage; they block publication, not
  authoring.
- `slug`: lowercase letters and digits in hyphen-separated groups, up to 100 characters. It
  cannot change once the page has been published.
- `title`: 2 to 160 characters.
- `entry_mode` is `hosted` (the default: Carusela serves the page) or `external`.
- **An external page is the club's own site.** Pass `external_url`: a public `https` address
  with no query, fragment, port or credentials. The content stays on that site, and its buy
  button links to the Carusela checkout on the club's own address. An external page takes no
  `seed_fields` and no `lesson_ids`.
- `seed_fields` (`title`, `cover`, `description`, `instructor`) and `lesson_ids` (up to 100,
  published lessons of that course only) copy course material into the first draft. A member
  tier page cannot seed from a course.

It never publishes. It returns the page id, `readiness`, `first_remediation` and the admin link.
Repeating the identical request with the same `request_id` returns the same draft.

Then author it:

```
get_sales_page             -> draft_document (the exact shape the edit takes back),
                              draft_revision, product, primary_offer, brand_kit,
                              authoring_context, versions, readiness
validate_sales_page_draft  -> the first thing wrong, by path; writes nothing
edit_sales_page_draft      { page_id, expected_draft_revision, document }
select_sales_page_primary_offer { page_id, offer_id }
```

The document is `{ schemaVersion: 1, slug, title, seo: { title, description, ogImageUrl },
blocks }`, with up to 100 blocks, each id unique. Validate before saving. Re-read after every
save, because the draft revision moves. Do not copy any `brand_kit` value into a block.

To switch an existing page between hosted and external, or change its external address:

```
set_sales_page_entry  { page_id, expected_draft_revision, entry_mode, external_url? }
```

It saves into the draft (a hosted page takes no `external_url`), so the switch reaches buyers only
through the publish below. Once an external page is published:

```
get_sales_page_external_integration  { page_id }
  -> checkout_url, snippets (HTML, React, Next.js), claude_code_instructions, connection
```

The buy link on the club's site is an ordinary `<a href="{checkout_url}">` in the same window,
never an iframe, popup, form post or script, and the snippets forward the campaign parameters.
When you can edit the club's site in this session, follow `claude_code_instructions` and deploy it
with the site's own tooling. `connection` shows the result of the last link test: a person starts
it on the page's admin screen, then clicks the buy link on the club's site in the same browser. An
ordinary buyer's click is not detected on its own.

#### A designed page

When the owner wants their own design rather than blocks (a long page, their own layout, many
images), write one HTML page and upload it:

```
save_sales_funnel_page  { page_id, html }  -> page_sha256
```

Then make it the draft's only block with `edit_sales_page_draft`:
`{ type: "customPage", variant: "full", props: { pageSha256 } }`. The rules, which the
`customPage` entry in `get_landing_page_catalog` states as well:

- One complete, self-contained, responsive HTML document, at most 256 KiB of UTF-8, with its own
  `<style>` and `<script>`. Upload images with `attach_media` and use their `https` URLs.
- A buy button carries `data-carusela-buy`, or is a link to `href="#carusela-buy"`. A click sends
  the buyer to this page's own Carusela checkout; nothing inside the page can charge.
- A video slot is an element with `data-carusela-video="<YouTube, Vimeo or Bunny URL>"` and an
  explicit size or `aspect-ratio`. Carusela draws the player over it; other providers get none.
- Size full-screen sections with `var(--carusela-viewport-height, 100vh)`, never bare `vh`,
  `dvh` or `svh`, which measure the frame and make the page scroll inside a fixed window.
- The page runs in a sandbox with an opaque origin: no cookies, storage, club API or form posts.
  `#section` links scroll, every other link opens a new tab. The browser title, description and
  share image come from the draft's `title` and `seo`, not from the HTML.
- The same HTML always has the same hash and a stored page never changes. To change the page,
  upload the new HTML and save the draft with the new hash.

Uploading needs the `commerce` capability; editing the draft needs `content_edit`.

Publishing:

```
publish_sales_page   { page_id }           -> published, version
rollback_sales_page  { page_id, version }  -> rolled_back, restored_from_version, version
```

`publish_sales_page` publishes the saved draft as a new version, in that call. It refuses while
the page is not ready and names what is missing, and a club that has not launched yet cannot
publish a public Sales page. When the saved draft is already the live version it answers
`already_live` and appends nothing, so a repeat is safe.

When the owner asks to see the page before it goes live, `preview_sales_page { page_id }` returns
the exact `candidate_document`, the `entry_url` and the `admin_url`, and writes nothing.
`preview_sales_page_rollback { page_id, version }` does the same for an older version.

To undo a publish, `rollback_sales_page` with a `version` from `get_sales_page`'s history restores
it in one request. A rollback appends the old document as a new version and edits nothing, so a
rollback can itself be rolled back. It has no repeat guard: a second identical call appends the
same version again, so call it once.

### 5. The Funnel

```
get_sales_funnel             { page_id }
list_funnel_followup_offers  { page_id, role?, page?, limit?, query? }
edit_sales_funnel_draft      { page_id, expected_draft_revision, expected_live_version, draft }
preview_sales_funnel         { page_id, expected_draft_revision, expected_live_version, draft }
publish_sales_funnel         { page_id }
rollback_sales_funnel        { page_id, target_version }
```

`get_sales_funnel` returns the current draft with its draft revision, the live version (0
when nothing is published), the primary offer, the offer's own single order bump, `repairs` and
every published version. `expected_draft_revision` is null when the page has no Funnel draft yet.

The draft is `{ orderBumps, steps, thankYou }`:

- `orderBumps`: 0 to 3 entries of `{ offerId, headline?, body? }`. A bump is paid inside the
  checkout, in the same charge as the primary offer.
- `steps`: 0 to 12 entries of `{ id, offerId, content?, onAccept, onDecline }`, in the order a
  buyer meets them. `id` matches `^[a-z0-9-]{1,40}$`. `onAccept` and `onDecline` are another
  step's id or `"thank_you"`, never the step's own id, and no arrangement may let a buyer reach
  the same step twice. `content` is `{ headline?, body?, acceptLabel?, declineLabel?,
  timerSeconds?, pageSha256? }`, at most 200, 2000, 60 and 60 characters, and a timer of 30 to
  3600 seconds. A field left out falls back to copy derived from the offer.
- `thankYou`: `{ heading, body?, nextAction?, pageSha256? }`. `heading` up to 200 characters,
  `body` up to 2000, and `nextAction` is `{ label, href }` where `href` is a path on the club's
  site or an `https` address. Without a `nextAction`, the button leads to what was bought: the
  course for a course purchase, the club home for a member tier.

**A designed step or Thank You.** Upload the owner's HTML with `save_sales_funnel_page` (the same
rules and the same `page_id` as a designed Sales page) and put the returned hash on the step's
`content.pageSha256` or on `thankYou.pageSha256`, keeping every other field of the draft. The page
is shown in the same sandbox above a fixed Carusela bar: on a step the bar carries the price, the
terms and the only accept and decline buttons; on the Thank You it carries the receipt and the next
action. A buy button inside the owner's HTML does nothing there, so do not tell the owner it will.
A hash this club never stored is refused with `page_not_found`.

**Upsell or Downsell is derived, not chosen.** A step reached only by declines is a Downsell;
every other step is an Upsell. Arrange `onAccept` and `onDecline` to get the shape the owner
wants, then check the kinds in the preview.

**Which offers the server accepts:**

- A step's offer costs more than 0 and is either a one-time course offer with no trial, or a
  member tier offer billed monthly or yearly. It is not the page's primary offer, not the primary
  offer's own order bump and not one of this Funnel's Order bumps. The same offer on two steps is
  refused. `list_funnel_followup_offers` returns exactly the offers a step may use, with their
  real publication flags; pick from it.
- An Order bump needs a primary offer that is a one-time course offer. The bump itself is a
  one-time course offer that costs more than 0, is not the primary offer, is not any step's offer
  and is not listed twice. When the primary offer allows instalments, the bump must allow
  instalments too, with a maximum at least as high. `list_funnel_followup_offers` with
  `role: "order_bump"` returns exactly the offers that pass these rules.
- Unpublished offers may sit in a draft. Publication needs them published.

Check a candidate with `preview_sales_funnel` before you save it. It writes nothing and returns
what each Order bump state charges at checkout and, for every step, its derived kind, where
accepting and declining lead, the amount, the access it grants and the consequences. Then save
with `edit_sales_funnel_draft`.

When the owner asked for the Funnel to go live, `publish_sales_funnel` publishes the **saved**
draft as a new version in the same turn. It reads the draft's bindings itself, so a draft or an
Offer that moves during the call refuses and nothing is published. It refuses while `repairs` is
not empty and when an offer or product in the Funnel is still unpublished. A publish consumes the
draft, so a repeat finds nothing staged and says which version is live. Then tell the owner what a
buyer meets on both branches of every step.

When the owner wants to walk the saved draft before it goes live, `confirm_sales_funnel_publish
{ page_id }` returns exactly what the publish would put live, both branches included, and writes
nothing.

To restore an older version, `rollback_sales_funnel` with a `target_version` older than the live
one restores it in one request, by appending it as a new version. A version whose follow-up offer
has since been unpublished is refused. Buyers already in a session stay on the version they
entered on. `confirm_sales_funnel_rollback { page_id, target_version }` walks that version first
when the owner asks, and writes nothing. A rollback has no repeat guard: a second identical call
appends the same version again.

A Sales page with no live Funnel cannot take buyers, hosted or external. When the owner wants no bumps and no
steps, publish a Funnel that is only a Thank You.

### 6. A/B tests (optional)

Only once the Funnel has at least two published versions; before that
`get_sales_funnel_experiments` reports `state: "unavailable"`.

```
get_sales_funnel_experiments   { page_id }
save_sales_funnel_experiment   { page_id, experiment_id | null, expected_revision | null, name, variants }
start_sales_funnel_experiment  { page_id, experiment_id, expected_revision, expected_live_version,
                                 variants_digest }
end_sales_funnel_experiment    { page_id, experiment_id, expected_revision, expected_live_version,
                                 variants_digest, winner_version }
```

`variants` is 2 to 4 entries of `{ version, weight }`: distinct published versions, whole weights
of 1 to 99 that sum to exactly 100. Only a draft test can be edited. One test runs per Funnel at a
time; each new checkout session is pinned to one variant, and a session in progress keeps its
version. Saving returns the test's `revision` and `variants_digest`, and
`get_sales_funnel_experiments` returns the `live_version`: when the owner asked for the test to
run, pass those to `start_sales_funnel_experiment` in the same turn. Start and end each apply in
that call. A start on a test that is already running, or an end on one that already ended, is
refused and names the status.

Ending with a `winner_version` publishes that version as the new live Funnel; ending with null
keeps the live one. Pass a winner only when the owner named it.

## Refusals and what they mean

| The tool says | It means | Do |
|---|---|---|
| needs the `commerce` (or `design_publish`) capability, or the seat does not hold it | wrong role, or wrong club | `list_my_clubs`; otherwise a person with that seat does it |
| the club is suspended and accepts no changes | the owner has to restore the club from the platform billing page | stop and tell the owner; retrying does not help |
| an existing draft requires its exact revision | `draft_id` sent without `expected_revision`, or the reverse | send both |
| `confirmation_token` is no longer accepted by any tool | an old instruction sent the retired argument | call again with the same arguments without it, after reading the current state |
| a selected draft is no longer staged at that revision | it was already applied, abandoned or revised, most often by the same apply arriving twice | `get_commerce_changes` and `get_audit_log`; do not stage the same change again on this alone |
| a live row changed after this draft was staged | the offer, tier or coupon was edited outside the draft | `get_commerce_catalog`, then revise the draft with its `manage_` tool (`draft_id`, `expected_revision`) or abandon it |
| the draft or the publication history moved | somebody saved since your read | `get_sales_page` or `get_sales_funnel`; call again only when what is saved is still what the owner asked for |
| this no longer matches this club's Funnel | the draft, an offer or an A/B test moved during the call, and nothing changed | `get_sales_funnel` (`get_sales_funnel_experiments` for a test); call again only when what is saved is still what the owner asked for |
| `already_live` from `publish_sales_page` | the saved draft is already the live version | nothing to do; edit the draft first to change the page |
| this test is already running, or already ended | the same start or end arrived twice | `get_sales_funnel_experiments` for its status |
| `request_id` already belongs to a different draft | the id was reused for a different request | reuse the original request unchanged, or pick a new id |
| slug already belongs to another Sales page | the slug is taken | choose another |
| `followup_offer_unavailable` on `steps` | that offer cannot be a step | pick from `list_funnel_followup_offers` |
| `order_bump_primary_incompatible` | the primary offer is not a one-time course offer | remove the bumps or change the primary offer |
| `order_bump_instalments_incompatible` | the bump allows fewer instalments than the primary offer | stage the bump offer with enough instalments, or pick another |
| `order_bump_offer_unavailable` | the bump offer fails the bump rules | pick another one-time course offer |
| `duplicate_step_offer`, `step_self_target`, `step_target_unknown`, `step_target_cycle` | the step list cannot be walked | fix the ids and targets |
| `max_steps_exceeded`, `max_bumps_exceeded` | more than 12 steps or 3 bumps | trim |
| `page_not_found` | a `pageSha256` this club never uploaded | `save_sales_funnel_page` first, then use the hash it returns |
| `funnel_products_unpublished` | an offer or product in the Funnel is unpublished | apply the commerce changes first, or publish through the commerce batch |
| nothing staged to publish, and the live version | a publish already consumed the draft, or none was ever saved | `get_sales_funnel` and `get_audit_log`; edit the draft only when the owner wants a change |
| `legacy_document_not_representable` | the old two-offer document shape was sent for a Funnel with bumps or more than two steps | send `{ orderBumps, steps, thankYou }` |
| a Sales page readiness code, such as `primary_offer_unpublished` or `landing_page_empty` | the page is not ready to publish | fix what it names; `get_sales_page` lists them all |

Every call, refused or not, lands in `get_audit_log` with `source: "mcp"`.

## When you are done

Read back, do not trust the receipt alone: `get_sales_page` for the published version and
`readiness.activate`, `get_sales_funnel` for the live version. Tell the owner what is live, what
a buyer meets on each branch, and what still stands between the page and its first buyer, most
often the CardCom connection or the club's launch.

What happens after payment is not configured here. A buyer who had no account is signed in
automatically in the browser that paid, and still receives the access email. An existing member
signs in the usual way.
