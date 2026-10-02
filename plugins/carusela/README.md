# Carusela for Claude Code

Run your club from Claude Code. This plugin connects the Carusela MCP and adds seven skills that
keep an agent inside what the surface actually allows, so it stops guessing at fields that are
refused on purpose and stops publishing things nobody reviewed.

## Install

```
/plugin marketplace add Carusela/carusela-plugin
/plugin install carusela@carusela
```

The MCP server comes with it. On the first Carusela tool call Claude Code opens the OAuth flow
against `https://carusela.com/api/mcp`; approve it with the platform account that owns the club.
Nothing else to configure, and no token to paste anywhere.

If your account reaches more than one club, every tool needs a `club_id`. Ask Claude to
"list my clubs" once and it will carry the right one from then on.

## The skills

| skill | when it fires |
|---|---|
| `club-orientation` | before the first write. The tool map, the three rings, and what MCP will never do |
| `seed-club-content` | importing or bulk-creating courses, recordings, tutorials, guides and agents |
| `brand-a-club` | colours, logo, favicon, social card, through the gated design flow |
| `gate-club-access` | deciding what each access tier reaches |
| `build-sales-funnel` | offers, coupons, a Sales page (blocks, a designed HTML page, or the club's own site) and its Funnel, each approved before it goes live |
| `audit-club-content` | after an import or before a launch: what is invisible and why |
| `tune-club-mentor` | making the club's AI assistant answer from the club's own material |

They fire from their descriptions, so in practice you say what you want and the right one loads.
You can also name one: "use audit-club-content".

## What this cannot do, by design

Prices, offers, coupons, trials and access tiers decide money and access, so they are written
in two steps: a tool stages a draft, a preview shows the exact proposal, and only the owner's
explicit approval applies it. Sales pages, Funnels and A/B tests publish through the same kind of
single-use token. Nothing charges a member, and the payment terminal is connected by hand in the
admin.

Feature flags are switched by the owner in the admin under "יכולות הקהילה". The navigation
rail is edited in the admin as well. The home page's block list is a design change, previewed
and published like colours.

There are no repository, deploy, DNS or domain tools here, and the platform stores no
corresponding credentials. Claude Code may already hold your own GitHub and Vercel sessions
locally; those never enter Carusela MCP.

## Things about this surface worth knowing

Not bugs, and the skills handle all of them for you. Listed because each one is a place where the
obvious guess is wrong:

- **Recordings, tutorials and guides join a category by carrying its exact name in `tags`**, not
  through a `category` field. Only groups, courses and AI agents have a real one. A near miss
  attaches nothing, so `manage_category list` reports a usage count per category and an empty one
  is visible immediately.
- **`tag` is printed on the card every member sees.** When moving courses from Schooler or another
  platform, the source id goes in `import_source` / `import_ref`, never in `tag` or `tags`.
  `seed-club-content` enforces this and `audit-club-content` finds the ones that got through.
- **`difficulty` is a closed set** on recordings, tutorials and guides: `מתחילים`, `ביניים`,
  `מתקדמים`, `כל הרמות`. Anything else is refused by name with the four values in the message.
- **The four library kinds have different create defaults.** A guide and an AI agent go live the
  moment they are created; a recording and a tutorial start as drafts. All four accept
  `is_published` on create, so staging a batch nobody has reviewed means passing it explicitly.

## Safety

Every call is written to the club's audit log with `source: "mcp"`, including failures and their
arguments, with emails redacted. Ask Claude to "show the audit log" to see what any session did.

Design changes cannot be published without a preview: `preview_design_change` returns a link a
human opens and a single-use token that `publish_design` spends. The skills require showing you
that link first.

Member names, emails and phone numbers come back only from the member tools, such as
`list_members` and `get_member`, which only the club's owner and admins can use. `get_member_stats`
gives counts only.
