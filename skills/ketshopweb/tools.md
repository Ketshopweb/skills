# Ketshopweb MCP tools

The server registers **66 tools**. You only see the ones your connection's scopes allow (see `connect.md`). Exact input schemas arrive with `tools/list`; op shapes, value ranges and worked examples come from the `guide` and `element_guide` tools. Do not invent fields.

## Markers

| Marker | Meaning |
|---|---|
| `read` | Changes nothing visible. (A few create bookkeeping, noted per tool.) |
| `draft` | Changes a draft only. The live site does not change until someone publishes. Never tell the user it is published. |
| `library` | Changes the media library at once (it has no draft) but does not change any page. |
| `LIVE` | Changes, or can change, the live site at once. Always ask the person first. |

Scopes: `R` = `pages:read`, `W` = `pages:write`, `M` = `media:write`, `P` = `site:publish`.
Count by scope: R 28 · W 20 · M 9 · P 9.

## The two-step confirm (all `site:publish` tools)

1. Call **without** `confirm`. Nothing happens. The answer is a summary of what would change plus a check value (`seq`, or `updatedAt` for `page_settings` / `page_delete`).
2. Show the summary to the person and ask.
3. Only if they agree, call again with `confirm: true` **and the same check value**.
4. If the draft or the page changed in between, the call is refused and nothing happens: start again from step 1 and ask again.

There is no "publish everything" tool. Pages, the site-wide draft, Global CSS, shared components and blog posts are each published by their own tool. If you do not have `site:publish`, tell the person to press "อัปเดตเว็บไซต์" (Update website) in the back office, which publishes everything pending.

## Guides

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `guide` | Rules, every page op with its shape, box values and ranges, the shop's languages. Read once at the start (text is Thai). | R | read |
| `element_guide` | Rules, limits and working example sources for Ai section Elements (TSX). Read before `element_check` / `element_write`. | R | read |

## Pages

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `page_list` | All pages: id, name, path, group, `hasDraft`, `locked`. Site-wide parts follow as entries with `kind: "global"` (`footer`, `blog-sidebar`) — use their id as `pageId`. | R | read |
| `page_read` | One page draft: the document (zones + sections), `seq`, `langs`, pending change count, op summary. Always read before `page_apply`. Reading a system page by key may create its row. | R | read |
| `page_create` | New empty custom page as a draft. `name`, optional `path`, `inMenu: true` also appends it to the main menu draft. | W | draft |
| `page_copy` | Copy a custom page's draft into a new draft page (new ids, not in any menu). Not for system pages or the footer. | W | draft |
| `page_settings_read` | A page's settings: name, URL per language, SEO per language, enabled, menus, `updatedAt`. | R | read |

## Sections and Elements of a page

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `page_apply` | Apply 1–100 ops (`section.*`, `node.*`, `words.set`, shared-component and Ai ops) to the current draft. All or nothing. Returns `seq`, `made` (new ids), `previewUrl`. | W | draft |

- Ops apply on top of the current draft; people may be editing the same page. The later write of the same field wins. `lockVersion` is accepted but ignored.
- Unknown fields or wrong types are an error, never silently dropped.
- Locked system pages cannot be edited.
- `pageId` is a numeric id or a system page key (`home`, `footer`, …) from `page_list`.
- Translating = `words.set` with `lang`. Only words are per language; structure, images and links are shared.
- Images are referenced as `media:<id>` (from `media_list`), never as URLs.

## Ai section / Element.tsx

An Ai section (or an AI Element inside a custom Section) is TSX source you write; the server validates and compiles it.

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `element_check` | Validate + compile source without saving. Returns `ok` + `tier`, or `problems [{rule, message, line, col}]`. `warnings` / `linkWarnings` never block but should be fixed. Needs no page. | R | read |
| `element_write` | Validate, compile, store, and point an Ai section (`sectionId`) or AI Element (`nodeId`) of the draft at the build. Failing source leaves the draft untouched. | W | draft |
| `element_read` | Source, name, tier, hash of the attached Element, plus its `meta` and current settings. | R | read |
| `element_settings` | Set one declared setting (products query, text, number, toggle, select, image, link, rich text) without rewriting the Element. Wrong values are refused, not clamped. `null` = default. | W | draft |

Usual order: `page_apply` (add a Section of type `ai`) → `element_guide` → `element_check` until it passes → `element_write`.

## Agent Files (the site as a folder of files)

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `files_pull` | One link (10 min) + shell command that downloads page drafts and shared components as a folder (tar.gz) with `AGENTS.md` and schemas. `since` = incremental pull. Stores a server snapshot (the merge base). | R | read |
| `files_status` | What changed on the server since a pull (`snapshotId`). No files sent. | R | read |
| `files_push` | One-time link (10 min) + command to send the edited folder back as drafts. All or nothing. `dryRun: true` = check only. | W | draft |
| `files_read` | For agents without a shell: the same folder as file contents in the answer. Returns `snapshotId`. | R | read |
| `files_write` | For agents without a shell: send changed files (path → content, `null` = delete) on top of a `files_read` snapshot. `dryRun` supported. | W | draft |

After a pull, read `AGENTS.md` in the folder first. A snapshot belongs to the connection that made it. Conflicts with newer server edits are reported as problems and nothing is saved.

## Media library

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `media_list` | Files (images, videos, PDFs), newest first, 60 per page; filters `q`, `folder`, `kind`; `after` for the next page. Each item has a `ref`. | R | read |
| `media_get` | One file plus `usedIn` (pages / components, draft or live). Check before deleting or replacing. | R | read |
| `media_folders` | All folders: id, name, parent. | R | read |
| `media_size` | Make (once) a smaller copy of an image and get its ref `media:<id>?s=<size>`. Max 16 sizes per file. | W | library |
| `media_upload_link` | One-time link (10 min) + shell command to upload ONE local file. The server never fetches URLs — download first. | M | library |
| `media_update` | Change a file's name, alt text or decorative flag. | M | library |
| `media_move` | Move files to a folder. Refs do not change. | M | library |
| `media_folder_create` | Create a folder (max 8 levels). | M | library |
| `media_folder_rename` | Rename a folder. | M | library |
| `media_folder_move` | Move a folder with its contents. | M | library |
| `media_replace_link` | One-time link to replace a file's content, same id and ref. Published pages that use it are rebuilt at once. | M | **LIVE** — confirm when in use |
| `media_delete` | Delete files. Cannot be undone. A live page that uses the file loses it at once. | M | **LIVE** — confirm when in use |
| `media_folder_delete` | `mode: "up"` moves contents to the parent (nothing deleted). `mode: "all"` deletes the folder with every file inside. | M | **LIVE** (`all`) — confirm when in use |

Confirm for these three is conditional: if any affected file is used by a page or shared component (draft or live) and you did not send `confirm: true`, nothing happens and the answer lists where it is used (`inUse`). Show that list, ask the person, then call again with `confirm: true`. Files not in use are replaced / deleted on the first call — so still ask before deleting anything the user did not name.

## Languages

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `lang_read` | Draft language list, `contentMain` (the language of words written without `lang`), languages live now, the shop's limit, `coverage: true` = translation progress per language. | R | read |
| `lang_ops` | `lang.add`, `lang.remove`, `lang.move`, `lang.set`, `lang.main`. A new language starts hidden so translations can be prepared. `dryRun` supported. | W | draft |

Ask the person before adding, removing, showing a language or switching the main language: these reshape the live site (URLs, redirects) once published.

## Menu and header

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `menu_read` | Draft main menu, parked items, header blocks, linkable pages, languages, limits, `seq`. | R | read |
| `menu_ops` | Batch of `menu.*` / `header.*` ops, all or nothing. `menu.put` / `header.put` replace whole blocks and need `seq` from `menu_read`; granular ops need no `seq`. `dryRun` supported. | W | draft |

Links to categories, products, blog posts and blog categories must use ids that exist (`catalog_lists`, `products_pool`, `blog_lists`).

## Site design, fonts, Global CSS

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `design_read` | Site style draft: colours, radius, fonts, text sizes, brand, buttons, font library, and the table of shop variables with the class to use in Element source. | R | read |
| `design_ops` | Batch of style ops (tokens, shape, type, brand, themes, site buttons), all or nothing. `themes.put` needs `seq`. `dryRun` supported. | W | draft |
| `font_add_google` | Add a Google Fonts family to the font library by name; the server stores the files as the shop's own. | W | draft |
| `font_upload_link` | One-time link (10 min) + command to upload font files as one new family (.ttf .otf .woff .woff2). | W | draft |
| `font_remove` | Remove a font from the draft library. If it is in use and `confirm` is not `true`, nothing is removed and the answer lists where; ask, then send `confirm: true`. Places that used it return to the default font. | W | draft — confirm when in use |
| `styles_read` | Global CSS draft (shared classes usable in Element source) + `lockVersion`. | R | read |
| `styles_write` | Replace the whole Global CSS draft. Send `lockVersion` from `styles_read`. `dryRun` supported. Write it before an Element that uses a new class. | W | draft |

Write Elements with the shop's classes from `design_read` (not fixed colours) so they follow later style changes.

## Product display (card, list page, detail page)

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `card_read` | Product card look + editable storefront words, presets, what cannot be set. | R | read |
| `card_ops` | `card.look`, `card.pd`, `card.badge`, `word.set`. Needs `seq` from `card_read`. | W | draft |
| `list_read` | Product list page settings (filters, layout, columns, paging, order), allowed values. | R | read |
| `list_ops` | `plist.set { patch }`. Needs `seq` from `list_read`. | W | draft |
| `detail_read` | Product detail page settings (layout, gallery, info pieces), allowed values, what the storefront does not draw yet. | R | read |
| `detail_ops` | `pdetail.set { patch }`. Needs `seq` from `detail_read`. | W | draft |

All three `*_ops` are all-or-nothing, support `dryRun`, and are refused as stale when the site draft changed since the `seq` you send: read again, merge, resend.

## Products and catalog lookups

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `catalog_lists` | Category tree, tags, brands, and running promotions (`deals`) with the ids product queries and promotion Elements need. | R | read |
| `products_pool` | Browse products to find ids (24 per page; `pool`: `all`, `featured`, `sale`; `search`, `cats`). | R | read |
| `products_query` | Run a product query exactly as the storefront would; returns the product cards (guest prices). | R | read |

Products themselves (names, prices, stock) cannot be edited through this server.

## Blog posts

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `blog_lists` | Blog category tree and posts with ids for blog Elements and menu links; optional tags. | R | read |
| `post_list` | Posts with state (shown / hidden / scheduled) and draft status. Filters: `slot`, `cat`, `status`, `q`, `page`. | R | read |
| `post_read` | One post: meta per language slot, `lockVersion`, body as one Section. Old-HTML blocks are read-only. | R | read |
| `post_check` | The publish check: blocking problems, warnings, final permalinks. | R | read |
| `post_create` | Create a new HIDDEN post (appears in the blog manager, not on the storefront). | W | draft |
| `post_apply` | Edit a post draft: body ops (`node.*`, `words.set`) and/or `meta`. Needs `lockVersion` from `post_read`; refused if someone edited in between. | W | draft |

## Preview

| Tool | Purpose | Scope | Marker |
|---|---|---|---|
| `page_preview` | A link (30 min) that shows the DRAFT on the real storefront, unindexed. Reloading shows the latest draft. Optional `lang` (must be a language already live). | R | read |

`page_apply`, `element_write`, `page_create` and `page_copy` return a `previewUrl` when one can be issued. While previewing, the whole site is shown as draft. Give the link to the user to review before anyone publishes.

## Publish and other LIVE actions (`site:publish`, two-step confirm)

| Tool | Step 1 (no `confirm`) returns | Step 2 (`confirm: true` +) | Effect |
|---|---|---|---|
| `page_publish` (`pageId`) | changes, `seq`, `nothingToPublish` | `seq` | Publishes one page's draft (also `footer`). Blog post bodies and locked pages are refused. |
| `site_publish` | changed parts, `seq` | `seq` | Publishes the site-wide draft: menu, header, site style, product display, languages. Every page changes. |
| `styles_publish` | changed classes, affected pages, `seq` | `seq` | Publishes Global CSS; published pages using changed classes are rebuilt. |
| `component_publish` (`key`) | live pages affected, `seq` | `seq` | Publishes a shared component; every published page using it changes. |
| `page_settings` (`pageId` + fields) | current values, `wouldBecome`, `updatedAt` | `updatedAt` | Changes name, `enabled`, URLs (`routes`), SEO. No draft — live at once. Old URLs redirect to new ones. |
| `page_delete` (`pageId`) | name, URLs, menus, `updatedAt` | `updatedAt` | Deletes a custom page from the live site. Cannot be undone. Not for system pages, home, footer. |
| `post_publish` (`postId`) | check, titles, permalinks, `seq` | `seq` | Publishes (or schedules) a post draft. Blocking problems = refused; fix with `post_apply`. |
| `post_unpublish` (`postId`) | titles, permalinks, `seq` | `seq` | Hides a post from the storefront. |
| `post_delete` (`postId`) | title, link per language, `seq` | `seq` | Deletes a post for good. Cannot be undone. You must show the title and link and ask. |

For `page_settings`, send only the fields to change, and send the same fields in both steps.
