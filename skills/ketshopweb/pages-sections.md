# Pages and Sections

Read this before calling any page tool. It is the map; exact field shapes, ranges and examples
come from the server: call the `guide` tool (no arguments; one Thai document; read it once per
session). `page_read` also returns an op summary in `ops`. If this file and `guide` disagree, `guide` wins.

## 1. Draft vs live

- Every editing tool writes a **draft**. The live storefront does not change until it is published.
- Publishing is normally done by a person: the "อัปเดตเว็บไซต์" (Update website) button in the
  back office publishes everything pending across the whole site in one step (all pages, header,
  menu, site style, footer, Global CSS, shared components).
- You can publish only if the shop granted `site:publish` (else the tools are not listed) — section 9.
- Never tell the person something is published unless a publish call returned success.
- People may be editing the same page while you work; they see your draft changes immediately.

## 2. What you can address as a "page"

`page_list` returns every entry. Use its `id` as `pageId` (a number for custom pages, a key
string for system pages and site-wide parts).

| Entry | How to recognise it | What you can do |
|---|---|---|
| Custom page | numeric `id`, `group` `menu` / `off-menu` | read, apply, preview, create, copy; settings / delete with `site:publish` |
| System page | `group: "system"`, key id such as `home`, `cart` | read / apply / preview unless `locked: true`; cannot be copied; name and URL are fixed for you |
| Page without a URL | `path: null` + `systemKey` (e.g. `not-found`, shown when a page is not found) | read / apply / preview; no settings, copy or delete |
| Site-wide part | `kind: "global"`: `footer` (bottom of every page), `blog-sidebar` (beside blog pages when the shop turns it on) | read / apply / preview / publish; not a page: no path, no SEO, cannot be created, renamed or deleted |

- `locked: true` = managed by the system, `page_apply` is refused. `hasDraft: true` = unpublished work.
- Footer and blog sidebar: nothing per visitor or per page, and **no product lists** (refused on save).
- Blog posts are not pages: use the `post_*` tools.
- URLs of product-list, product-detail and blog pages are set per shop and language: never
  hard-code `/products`, `/blog` or a product URL; link by page reference (see elements.md).

## 3. The page document

`page_read` returns `doc = { zones, sections }` plus `seq`, `langs`, `locked`, `pendingChanges`.

A Section:

```
{ id, type, name, zone, hidden?, hideOn?, data, lang?, box?, screens?, nodes? }
```

| Field | Meaning |
|---|---|
| `type` | Section type (section 4) |
| `zone` | which zone of the page it sits in (section 5) |
| `hidden` | `true` = not on the live site on any screen |
| `hideOn` | screens it is hidden on: `"desktop"`, `"tablet"`, `"mobile"` |
| `box` | the Section's box on desktop (section 6) |
| `screens.tablet` / `screens.mobile` | only the box keys that screen overrides. Missing key = same as desktop; `null` = no value on that screen. Each screen inherits from desktop only |
| `data` | words of the **main language**: `data["<elementId>.<word>"] = "text"` |
| `lang[code].data` | words that override `data` for that language only |
| `nodes` | the Element tree (custom Sections; slots of Ai sections) — see elements.md |

Sections render top to bottom in array order, grouped by zone.

## 4. Section types

| `type` | What it is | Notes |
|---|---|---|
| `custom` | A Section the shop arranges itself: a box holding Elements | The normal choice. `section.add` with `grid: true` starts it as a 3-column grid |
| `ai` | A frame whose content is a component written by the assistant | Content is written with `element_guide` → `element_check` → `element_write`, not with `node.*`. Its top level holds only **slots** (see elements.md). Clear it with `ai.build` (`build: null`) |
| placed shared component | A Section that has `use: "<component key>"` | Added with `section.use`. No box / nodes of its own; only its props, name and visibility can change. `use.detach` turns it into ordinary content of this page |
| system block | e.g. `not-found`, `blog-index`, `blog-post`, `product-list`, `product-detail`, `member-login`, `member-register`, `no-access` | Put there by the system; **cannot be added with `section.add`** |

System block restrictions: cannot be removed, moved, hidden, duplicated or renamed; takes no
Elements and no words. Only its box keys `contain`, `pad`, `fill` change (`section.box`). Its
texts are system words a person edits in the back office, not Section words.
The editor's Section presets "products" and "Catalog Section" are not available yet.

## 5. Zones

- Ordinary pages have one zone, `main`. Pages with a system block (404 page, blog pages, …)
  have three: `top` (insertable), `system` (closed), `bottom` (insertable) — see `doc.zones`
  (`insert` = Sections can be added).
- `section.add` / `section.use` accept `zone` (must be insertable). Omitted = the zone of `after`
  if insertable, else the first insertable zone. `section.place` accepts a target `zone`.
  `section.move` at a zone edge jumps to the next insertable zone.

## 6. The box (overview)

The same box model is used by Sections and Elements; each kind accepts a subset. Set it with
`section.box` / `node.box` and a `device` (`"desktop"` = base for all screens, `"tablet"`,
`"mobile"` = override for that screen). In `patch`, `null` removes a key; `reset: "all"` or
`reset: [keys]` on tablet / mobile returns to the desktop value.

| Group | Keys | Plain meaning |
|---|---|---|
| Layout | `layout`, `grid`, `dir`, `wrap`, `align`, `distribute`, `gap`, `clip`, `order` | row / column or grid, alignment, spacing between children, per-screen child order |
| Spacing | `pad`, `mar` (Elements only), `vmar` (Sections only) | inner padding; outer margin; a Section's top / bottom gap, which replaces the site-wide gap at that join |
| Size | `w`, `h`, `minW`, `maxW`, `minH`, `maxH`, `contain` (Sections only) | px, `"fit"`, `"fill"` or `"50%"`; `contain` = content width: `"site"`, `"full"` or px |
| Fill | `fill` | colour, gradient, or image |
| Look | `opacity`, `radius`, `border`, `shadow` | transparency, corner rounding, border, shadow |
| Grid cell | `cs`, `rs`, `self`, `selfX` | for a piece inside a grid parent: columns / rows spanned, position in the cell |
| Sticky | `sticky` | Elements only (some kinds); stays on screen while its group scrolls past |

- Colours are `{"token": "<id>"}` (site colours such as `primary`, `text`, `surface`, `line`,
  and the dark set `dark`, `on-dark`, …) or `{"hex": "#RRGGBB"}`. **Prefer tokens**: they follow
  the site style. The token list is in `guide`.
- Image fill: `fill: { kind: "image", ref: "media:<id>", mode?, pos?, fixed?, size?, tint?, tintAlpha? }`.
  The `ref` comes from `media_list` (or an upload). **Never a URL.** A deleted file draws no image.
- Out-of-range or malformed values are **rejected** (the message gives the range), never rounded.

## 7. The read → apply loop

`guide` (once) → `page_list` → `page_read({ pageId })` → `page_apply({ pageId, ops })` (1–100
ops) → open the returned `previewUrl` (or call `page_preview`) → repeat. Rules of `page_apply`:

- **All or nothing.** The first invalid op means nothing is saved; the error names the op number
  and the reason. Fix and resend. Unknown op names and unknown fields are errors (typos are caught).
- Ops are applied on the **current** draft. Different Elements / fields merge with other editors'
  work; for the same field the later write wins. No version needs to be sent (`lockVersion` is
  accepted but ignored).
- "Section / Element not found" usually means a person removed it meanwhile: `page_read` again,
  then build new ops. Do not guess and resend.
- Result: `{ ok, seq, made, pendingChanges, previewUrl?, previewExpiresAt?, refWarnings? }`.
  `seq` = draft sequence number after the write. `refWarnings` lists category ids that do not
  exist in the shop (not blocking).
- **Ids.** New Sections / Elements get generated ids, returned in order in `made` (`s…` =
  Section, `e…` = Element). You cannot reference a generated id inside the same batch — unless
  you pass your own `id` on `section.add` / `node.add` (a–z, 0–9, `_`, `-`, max 64 chars, unique
  in the whole page). Supplying ids lets one batch add and then fill.
- Limits: 100 ops per call; 400 Elements per Section.

Preview: `page_preview({ pageId, lang? })` returns `{ url, expiresAt }`. The link shows the
**whole site as draft** on the real storefront with a "draft" banner, is valid 30 minutes, and
always shows the latest draft on reload (no need to request a new one after each edit). `lang`
works only for languages already shown on the live site. For `footer` the link opens the home
page; for `blog-sidebar` the blog list.

## 8. Creating pages and page settings

| Tool | Does |
|---|---|
| `page_create({ name, path?, inMenu? })` | New empty custom page as a draft (only when the shop asks; pages are limited). `path` = main-language URL (omitted = made from the name). `inMenu: true` also appends it to the main-menu draft. Returns `{ id, key, path, menuNode?, seq?, previewUrl? }` |
| `page_copy({ pageId })` | Copies a custom page's draft (new ids) to a new draft page named "… (สำเนา)" with a free URL; not in any menu |
| `page_settings_read({ pageId })` | Name, URL per language, SEO per language, enabled, menu placement. Read-only |
| `page_settings(…)`, `page_delete(…)` | Need `site:publish`. Settings have **no draft**: changes hit the live site at once (old URL redirects to the new one; disabled = 404; delete cannot be undone from the back office) |


## 9. Publishing (only with `site:publish`)

`page_publish({ pageId })` (also `"footer"`, `"blog-sidebar"`), `site_publish()` (menu, header,
site style, product display), `styles_publish()` (Global CSS), `component_publish({ key })`.
There is no single "publish everything" call for you.

Every publish / settings / delete tool is **two-step**:

1. Call without `confirm` → the list of what would change plus `seq` (or `updatedAt`); nothing happens.
2. Show the list to the person and ask. Only after an explicit yes, call again with
   `confirm: true` + the same `seq` / `updatedAt`. Edited in between = refused: redo step 1.

Never send `confirm: true` without the person's answer; "fix this page" is not permission to
publish. `nothingToPublish: true` = draft equals live.

## 10. Languages

- `page_read` → `langs`: `[{ code, name, main, shown? }]` — every language in the shop's draft,
  including ones not shown on the site yet. Only these codes are valid as `lang`.
- Words without `lang` belong to the main language. `words.set` with `lang` writes that
  language's override; a key not set falls back to the main-language word; `null` clears it.
- Structure (Sections, Elements, boxes, order, images, links) is shared by all languages. To
  translate, change **words only**. Image `alt` is translatable; `src` is not.
- To translate a site: `lang_read({ coverage: true })` (heavy; shows what is untranslated per
  page) → `page_read` → `page_apply` with `words.set` + `lang` per Section.
- Site languages change with `lang_ops` on the site draft (`lang.add`, `lang.remove`, `lang.move`,
  `lang.set`, `lang.main`). **Ask the person first**: URLs and the language switcher change once
  published. A new language starts hidden so translations can be prepared.
- After `lang.main` (switch main language) and before it is published, words sent without `lang`
  still belong to the *old* main language. `lang_read` → `contentMain` tells you which language
  that is; in that state always pass `lang` explicitly.

## 11. Op names (for `page_apply`)

| Group | Ops | Meaning |
|---|---|---|
| Section | `section.add` | Add a Section (`type`, `after`, optional `id`, `grid`, `onlyOn`, `zone`) |
| | `section.remove` · `section.duplicate` | Delete · copy after itself (all new ids in `made`) |
| | `section.move` · `section.place` | One step up / down (`by: -1 \| 1`) · put before another Section (`before: id \| null` = end) |
| | `section.set` · `section.box` | Name, `hidden`, `hideOn` · the Section box on one device |
| Element | `node.add` · `node.remove` · `node.duplicate` | Add to a Section or group · delete (with its words) · copy |
| | `node.move` · `node.nudge` | Move to another parent / Section / index (per-screen order with `device`) · one step within its parent |
| | `node.set` | Name, `hidden`, `hideOn`, `link`, `locked` |
| | `node.box` · `node.cfg` | Box · the kind's settings, on one device |
| | `node.wrap` · `node.unwrap` | Put an Element into a new group · dissolve a group |
| | `node.kind` | Switch between `image` and `video` only |
| Words | `words.set` | Set / clear words of a Section (`lang` optional) |
| Component | `section.use` · `node.use` | Place a shared component as a Section · as an Element |
| | `use.props` · `use.detach` | Set props of a placed component (`lang` optional) · detach it into page content |
| AI | `ai.build` | `build: null` clears an Ai section / AI Element (writing is done with `element_write`) |
| | `ai.settings` | Set one setting of an Ai section / AI Element (prefer the `element_settings` tool) |

Site-level ops are **not** `page_apply` ops; they belong to their own tools: `menu.*` /
`header.*` (`menu_ops`), `lang.*` (`lang_ops`), `tokens.set` / `shape.set` / `type.set` /
`brand.set` / `buttons.*` / `btnkind.*` / `themes.put` (`design_ops`), `card.*` / `word.set`
(`card_ops`), `plist.set` (`list_ops`), `pdetail.set` (`detail_ops`). Shared component
definitions are edited as files (`files_pull` → edit → `files_push`), not with `page_apply`.
