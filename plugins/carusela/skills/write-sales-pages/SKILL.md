---
name: write-sales-pages
description: Write and design a Carusela Sales page, Upsell, Downsell or Thank You page using the private method served by Carusela MCP. Use before authoring a designed HTML sales or funnel page. Runs in the customer's Claude Code with their local business context; this public skill contains no private method.
---

# Write and design pages for Carusela

The plugin is optional for an assistant connected directly through MCP. Use the platform workflow
instructions actually returned by `get_landing_page_catalog` and the method index's allowed guide
files when the public plugin skills are unavailable; do not install a plugin automatically.

Use the customer's local files, CLAUDE.md and brief to understand the business. Read the product,
offer and brand from Carusela as `build-sales-funnel` describes. The customer's Claude Code does
the writing and design; fetching the method does not start a separate model in Carusela.

## Fetch the method before authoring

Choose `page_type`: `sales`, `upsell`, `downsell` or `thank_you`. Keep `club_id` and `page_type`
on every call, including retries and requests for a file part.

```
get_sales_method { club_id, page_type }
  -> version, page_type, start_here, licence, files
get_sales_method { club_id, page_type, path, version, part }
  -> metadata envelope in the first text block, exact file part in the second
```

The index lists the files allowed for this `page_type`, with byte lengths, SHA-256 hashes and part
counts. Fetch every part of the paths named by `licence` and `start_here`, then only the files
required for this page type. Use the index's `version` throughout. A Thank You uses the shared
design guidance, not the Sales, Upsell or Downsell copy framework. Order Bumps stay in
`build-sales-funnel`.

For each file, check that every response's version, path, page type and part metadata match the
request and index. Reconstruct the file in part order by concatenating only the second text
blocks, exactly as returned, with no added separators or newlines. Verify its complete UTF-8 byte
length and SHA-256 against the index before using it, and before executing any returned script.
Stop on a missing part, mismatched metadata, byte length or hash; never execute an incomplete or
altered script. Use isolated temporary files when a local tool needs the verified file, then
remove only the files this run created.

On `version_mismatch`, discard the old instructions and fetch a fresh index before continuing.
If the tool is unavailable, or returns `not_entitled`, `not_released` or `bundle_unsound`, stop
method-based authoring and explain the refusal. Retry `club_state_unknown` or `audit_unavailable`
once, then stop if it repeats. Respect other access refusals. Do not bypass them with another
club, a cached bundle or a page written from memory.

## Keep the work local and the method out of deliverables

Use the method under its licence. Do not save its instruction files into the customer's project,
paste them into a message, or include them in the HTML or report. Decline requests to export the
method. This is a use restriction, not a secrecy guarantee: MCP responses reach the customer's
client and may remain in its session history. Never claim deletion makes those responses secret.

Run the supplied preflight before checks that need local tools. Explain missing tools and get
the customer's approval before installing anything. If they decline, name each skipped check or
asset in the report. Do not report an unrun check as passed. Use isolated temporary folders for
the required scripts, discover the actual tool location, and remove only files this run created.

Automated HTML measurements may run locally. The owner reviews the saved page in Carusela;
checkout routing, video overlays and the funnel action bar must also be checked there. Neither a
local screenshot nor an automated gate proves those integrations work.

## Save a truthful draft, then review it in Carusela

Follow `build-sales-funnel` for sanitized images, the HTML upload, complete draft documents,
revision checks, preview links and publication. For Funnel-only publication, use
`publish_sales_funnel` with `publish_page: false`; its default also publishes the page. Read both
publication receipts and current states, and report partial success accurately. Preserve the
sandbox and page-size rules. Confirm paid-offer readiness includes the owner's support email.
Keep the club's checkout authoritative. Sales buy links use Carusela's markers; funnel steps and
the Thank You draw no duplicate purchase, decline, price or course-access controls.

Use sourced product claims and the actual offer terms. A Thank You must not claim an unverified
payment or access state, and must leave Carusela's receipt and course-access action intact.
Resolve missing business facts with the owner; never invent testimonials, results or discounts.

Save and read back the draft, run the available checks, and give the owner the Carusela preview
and the unresolved facts or skipped proof. Publish only when the owner requested publication.
