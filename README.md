[![Carusela for Claude Code: Your club. One conversation.](assets/brand/social-preview.png)](https://carusela.com)

# Carusela for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-163300?style=flat-square&labelColor=0e2200)](LICENSE)
[![Claude Code plugin](https://img.shields.io/badge/Claude_Code-plugin-9fe870?style=flat-square&labelColor=163300)](#install)
[![Visit Carusela](https://img.shields.io/badge/carusela.com-Visit-9fe870?style=flat-square&labelColor=163300)](https://carusela.com)

[Install](#install) · [Skills](#why-a-plugin-and-not-just-the-mcp) · [Safety](#safety) · [Contributing](#contributing)

Run your [Carusela](https://carusela.com) club from Claude Code. Build courses, upload a library,
brand the club, set access tiers, build sales pages and funnels and configure its AI mentor, by
talking to Claude.

## Install

```
/plugin marketplace add dangogit/carusela-plugin
/plugin install carusela@carusela
```

Both are typed **inside Claude Code**, not in a terminal. The MCP connection comes with the
plugin, so there is no server address to paste and no token to copy. The first time you ask
Claude to do something in your club, a browser opens and you approve it with the Carusela
account that owns the club. Once.

## Why a plugin and not just the MCP

The connection on its own gives Claude the tools. It does not tell Claude how the surface is
shaped, so it guesses, and several of the natural guesses are quietly wrong: content created live
when you wanted a draft, a category that looks attached and is not, a tier that does not exist.

The seven skills are the part that stops that.

| skill | what it is for |
|---|---|
| `club-orientation` | the map, and which refusals are correct so Claude stops trying to route around them |
| `seed-club-content` | importing or bulk-creating content without losing fields or publishing early |
| `brand-a-club` | colours, logo, favicon and social card, through the preview-then-publish gate |
| `gate-club-access` | what each access tier reaches |
| `build-sales-funnel` | offers, coupons, a sales page (blocks, your own designed HTML page, or your own site) and its funnel, each approved by you before it goes live |
| `audit-club-content` | what got created successfully and is still invisible |
| `tune-club-mentor` | making the club's AI assistant answer from your own material |

They load from their descriptions, so in practice you say what you want. You can also name one:
"use audit-club-content".

## What it will not do

Prices, offers, coupons, trials, instalments and member tiers decide money and access, so Claude
never changes them in one step. It stages a draft, shows you the exact proposal and a buyer
preview, and applies it only after you say yes. Sales pages, funnels and A/B tests go live the same
way. Claude never charges or refunds a member, and it cannot connect your CardCom terminal: you do
that yourself in the admin, under payments.

Feature flags are yours to switch in the admin under "יכולות המועדון", and the navigation rail is
edited in the admin as well. Claude does neither.

There are no repository, deploy, DNS or domain tools here, and Carusela stores no such
credentials. Claude Code may already hold your own GitHub and Vercel sessions on your machine;
those never enter this connection.

## Safety

Every call is written to your club's audit log with `source: "mcp"`, including the ones that
failed and the arguments they carried, with email addresses redacted. Ask Claude to "show the
audit log".

Design, commercial, sales page and funnel changes cannot go live without a preview: staging
returns what would change and a single-use token that publishing spends, and the skills require
showing it to you first.

Member names, email addresses and phone numbers come back only from the member tools, such as
`list_members` and `get_member`, which only the club's owner and admins can use. Member statistics
are counts only.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- A Carusela club, and the platform account that owns it

## Contributing

Found a missing workflow or an unclear instruction? [Open an issue](https://github.com/dangogit/carusela-plugin/issues)
with what you asked Claude to do and what you expected. Leave out tokens, member data and
private club content.

Skill improvements and documentation fixes are welcome as pull requests. Each skill lives in
[`plugins/carusela/skills`](plugins/carusela/skills). Keep changes focused and preserve the
preview and approval steps for actions that publish content or change money and access.

For articles, demos and integrations, the [brand assets](assets/brand) include the logo,
wordmark, banner source and a social preview image.

## Licence

MIT. See [LICENSE](LICENSE).
