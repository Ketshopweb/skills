# Elements

Elements are the pieces inside a `custom` Section (and inside the slots of an Ai section).
This file is the map. Exact settings, ranges, defaults and a worked example per kind come from
the `guide` tool (no arguments) — read the heading for the kind you are about to use. When this
file and `guide` disagree, `guide` wins.

## 1. The tree

- `Section.nodes` is an ordered tree. A node: `{ id, kind, name?, hidden?, hideOn?, link?, box?,
  screens?, cfg?, cfgOn?, children? }`.
- Only containers have `children`: `group`, `tabs` (children are `tab` only) and `tab`.
- Groups nest up to **4 levels** inside a Section (the Section's own box is not counted; a `tabs`
  and its `tab` panes count as one level). Max **400 Elements per Section**.
- Ids are unique across the whole page. Pass your own `id` on `node.add` (a–z, 0–9, `_`, `-`,
  ≤ 64) to reference it later in the same batch; otherwise read it from `made`.
- `node.add` creates the Element with sensible defaults (some kinds come with starter items and
  words). `parentId` omitted = directly in the Section's box. `onlyOn` = show on that screen only.

## 2. Three places a value can live

| Where | Set with | Holds |
|---|---|---|
| `box` (+ `screens`) | `node.box` + `device` | size, spacing, fill, look — each kind accepts only some box keys (listed per kind in `guide`) |
| `cfg` (+ `cfgOn`) | `node.cfg` + `device` | the kind's own settings. `desktop` is the base; `tablet` / `mobile` override. `null` in `patch` = back to default |
| words | `words.set` on the **Section** | all text and file references, key `"<elementId>.<word>"`; per language with `lang` |

- Some settings are **shared across screens** (item lists, sources, most behaviour). Sending them
  with any `device` writes the same single value — set them with `device: "desktop"`.
- Word kinds: plain text (with a max length), rich text, and **media** (`media:<id>` or `""`).
- List Elements keep their items in `cfg.items` as `[{ "id": "x" }, …]` (id: letters, digits,
  `_`, `-`, 1–24 chars, unique) and item words at `"<elementId>.<itemId>.<word>"`.
  To add an item: send the whole new `items` list plus the new words in the same batch.
  To reorder: resend `items` in the new order. Removing an item from `items` deletes its words in
  every language. A word for an item not in `items` is an error.

## 3. Media references

- Files are referenced as `media:<id>` — find them with `media_list`, or upload
  (`media_upload_link`, needs the `media:write` permission; otherwise ask the person to upload).
- **Never put a URL** where a media ref is expected (rejected, including images from other sites).
  An image you fetched or generated must be uploaded to the library first.
- A smaller rendition: `media_size({ id, size })` returns a ref like `media:<id>?s=400x300`; use
  the returned ref. Pick a size matching where it is drawn.

## 4. Links

`node.set` with `link` works on `group`, `text`, `image`, `button`, `icon`:

```
{ "to": "page" | "section" | "url" | "email" | "tel", "value": "…", "newTab"?: true, "off"?: true }
```

| `to` | `value` |
|---|---|
| `page` | a page id from `page_list` (number, or a system key such as `cart`). The URL is resolved at publish, per language — this is how to link to product-list, blog and other system pages |
| `section` | id of a Section on this page |
| `url` | `https://…`, or a library file `media:<id>` (e.g. a PDF) |
| `email` / `tel` | an address / a phone number (`+` allowed, ≥ 6 digits) |

- `newTab` only for `page` / `url`. `off: true` keeps the target but disables the link.
  `link: null` removes it.
- **No nested links**: nothing inside a linked group can carry a link, and a group cannot be
  linked if something inside already is. `faq`, `accordion`, `tabs`, `slide` cannot sit inside a
  linked group.
- `slide` and `marquee` have no node link; each item has its own `link` in `cfg.items[]`.
- Links to a category, product, blog post or blog category by id exist in **menus**
  (`menu_ops`), not as a node link. On a page, use the data Elements below (their cards link by
  themselves) or a `page` link.
- If the target page's URL changes, the linking page must be published again.

## 5. Kinds that exist today

| `kind` | What it is | Words | Notable rules |
|---|---|---|---|
| `group` | Box that holds other Elements (row / column or grid) | — | Full layout box. Can carry a link |
| `text` | Text block | `text` (rich) | See section 6 |
| `divider` | Line | — | cfg: `style`, `thick`, `color`, `len`, `align` |
| `image` | Picture | `src`, `srcT`, `srcM` (media), `alt` | See section 8 |
| `video` | Library file, YouTube or Vimeo | `src`, `poster` (media), `title` | `cfg.source` + `cfg.vid` = the video **id, not a URL**. Autoplay is always muted. Not drawn until it has a file / id |
| `button` | Button | `text` (plain, ≤ 80) | See section 7 |
| `icon` | Icon from the built-in sets | `label` | `cfg.icon` = a name from the list in `guide` (unknown name → error with suggestions). No SVG / URL. Give linked icons a `label` |
| `faq` | Question / answer list with optional title | `title`, `sub`, `<item>.q`, `<item>.a` (rich) | List Element. Starts with items `q1`, `q2` |
| `accordion` | Collapsible list | `<item>.q`, `<item>.a` (rich) | List Element, like `faq` without title |
| `tabs` + `tab` | Tab set; each `tab` is a pane holding Elements | `<tabId>.label` | ≤ 12 tabs. `node.add` `tabs` **without** an id creates 3 filled tabs (`made` order is fixed); **with** an id creates an empty set — add `tab` children yourself. A `tab` cannot be hidden, linked or moved to another set; no `tabs` inside a `tab` |
| `slide` | Image slider | `<item>.src`, `<item>.srcM` (media), `<item>.alt` | List Element; per-item `link`, `tint`, `alpha`, `blur`. Slides without an image are not shown. Height via `cfg.fixH` / `cfg.h`, not the box |
| `countdown` | Countdown clock | `uD`, `uH`, `uM`, `uS` (unit labels) | `cfg.at` = target time in **epoch milliseconds** (not seconds, not a date string). Not drawn until set |
| `marquee` | Endless scrolling strip of texts / logos | `<item>.text`, or `<item>.src` (media) + `<item>.alt` | List Element, ≥ 1 item; item `kind: "img"` = image; optional `icon`, `link` per item |
| `compare` | Before / after image with a draggable divider | `picB`, `picA` (media), `lbB`, `lbA` | Needs both images to be drawn. All settings shared across screens. Height follows `cfg.ratio` |
| `table` | Data table | `<rowId>.<colId>` (plain, ≤ 500) | `cfg.rows` / `cfg.cols` are id lists; **column ids start with `c`**. New rows / columns and their words must be in the same batch, `words.set` **before** `node.cfg`. Empty cell = no word (do not write `""`) |
| `html` | Shop-supplied HTML | `code` | **JavaScript is stripped** (`<script>` shows as text, `on…` attributes and `javascript:` links removed). Prefer system Elements or an Ai section when possible |
| `facebook` | Facebook Page plugin | — | `cfg.url` = canonical page URL. Inside look belongs to Facebook; the Element takes no box look. All settings shared across screens |
| `product` | Product Highlight: product cards from a query | — | `cfg.from` (`latest`, `cat`, `tag`, `brand`, `promo`, `random`, `pick`, …), `count`, `sort`, columns, slider. Ids from `catalog_lists` / `products_pool`; test with `products_query`. Card look = the site's product card |
| `flashsale` | Product Promotion: one deal (header, timer, its products) | `head`, `logo` (media) | `cfg.kind` (`flash` / `gift` / `coupon`) + `cfg.promo` (deal id from `catalog_lists`). A deal that applies to all products needs `cfg.all: true` or no cards are drawn |
| `blog` | Blog Highlight: article cards | — | `cfg.from`: `latest` / `pick` / `cat`; ids from `blog_lists` |
| `prodcats` | Product category tiles with images | `<item>.pic` (media), `<item>.name` | `cfg.from`: `top` / `sub` / `pick`; items are `{ id, ref: <category id>, pick? }` |
| `category` | Category list / tree | `title` | Items `{ id, ref }` (`[]` = all). `cfg.of: "blog"` = blog categories |
| `ai` | AI Element: a small component written by the assistant, placed among other Elements | — | Content written with `element_guide` → `element_check` → `element_write`; cleared with `ai.build` (`build: null`); settings with `element_settings` |
| placed component | A node `{ id, kind: "use", use }` | props at `<id>.<prop>` | Added with `node.use`; only props, name, visibility and its own size change. `use.detach` to edit the inside |

Shop data (product names, prices, stock, category names and counts, deal times, article titles)
is read live on every page view. **Never copy it into words** or hard-code it.

Not available yet (shown as "coming soon" in the editor; there is no `kind` to add): map, form,
storefront search, AI Search, storefront coupons, Progress Bar Discount, DBD Account.

### Where kinds are refused

| Place | Refused |
|---|---|
| Slot of an Ai section | `product`, `flashsale`, `blog`, `slide`, `countdown`, `marquee`, `compare`, AI Elements, placed components |
| Footer and blog sidebar | `product`, `flashsale` (anything listing products). `blog`, `category`, `prodcats` are allowed |
| Per page | limited number of distinct product queries, deals and blog sets (numbers in `guide`); exceeding = nothing saved |

### Slots of an Ai section

An Ai section's top-level nodes are only **slots**: groups carrying `slot: "<name>"` that match
a slot declared in the section's source. Create one with `node.add { kind: "group", slot }`.
A slot still declared in the source cannot be removed, moved or duplicated. Inside a slot use
ordinary Elements (minus the refused kinds above). Use slots when the shop wants to edit texts /
images / buttons on screen themselves. Details: `element_guide`.

## 6. Text

- Words: `<id>.text` = the plain text (this is what layers, translation and you read).
  Optional `<id>.text~doc` = a rich document as a **JSON string** for inline styling.
- Plain text is enough for most cases: send only `<id>.text`.
- The rich document accepts only: `doc` › `paragraph` | `bulletList` / `orderedList` ›
  `listItem` › `paragraph`; inline `text`, `hardBreak`; marks `bold`, `italic`, `underline`,
  `strike`, `textStyle` (colour, weight 400–700, font size, font family) and `link`
  (`{ "type": "link", "attrs": { to, value, newTab? } }`, same shape as a node link).
  Anything else is rejected. There are no headings inside the document.
- The document's text must spell the same words as `<id>.text`. When you rewrite the plain text
  and are not sending a matching document, clear the old one (`"<id>.text~doc": null`).
- Whole-block style lives in `cfg`: `tag` (`p`, `h1`–`h6`; shared across screens), `role`
  (`body` / `strong` / `small` — size follows the site style), `size` (px; only when you
  deliberately want a fixed size; `null` = back to the role), `weight`, `color`, `align`, `lh`,
  `font`, `lines`, …
- Use `tag` for semantic headings; use `role` before `size` so text follows the site style.
- Inside a linked group, inline links are drawn as plain text.
- Rich answers of `faq` / `accordion` follow the same rules (`<id>.<item>.a` + `…a~doc`).

## 7. Button

- `cfg.look`: `primary` (default, solid) or `secondary` (outline) — the look comes from the
  site style, so one site-level change restyles every button. A shop-defined type is
  `cfg.look: "k-xxxxxx"` (ids from `design_read` → `buttons.kinds[]`; only existing ids).
  A type that was deleted is drawn as the secondary button.
- **Custom look**: setting box `fill` / `border` / `radius` / `shadow` or `cfg.color` makes the
  button on that screen use its own look instead of the type's (unset parts = none). Return to
  the type by setting those (and `cfg.own`) to `null`. Prefer types over custom looks.
- `cfg.size`, `cfg.weight`, `cfg.font` are optional; unset = the type's values. Per screen.
- The button box accepts only `w`, `fill`, `opacity`, `radius`, `border`, `shadow`
  (`w: "fill"` = full width).
- Link via `node.set`. A button without a link is just a label.

## 8. Image

- `<id>.src` = `media:<id>` (or `""` = no image). `<id>.srcT` / `<id>.srcM` = a different file on
  tablet / mobile (unset = `src`).
- `<id>.alt` = description for this placement (empty = the file's own alt; decorative files have
  none). Translatable per language.
- `cfg.ratio` (`4:3`, `3:4`, `1:1`, `16:9`, `9:16`, `4:5`, `file`), `cfg.fit` (`cover` /
  `contain`), `cfg.big` (click to enlarge), `cfg.zoom` (zoom on hover). A linked image does not
  enlarge.
- Only when the shop itself supplies an image link: `cfg.source: "link"` + `cfg.url` (https
  only). Otherwise always the library.
- The image Element is always lazy-loaded. For the first image on screen (hero) build an Ai
  section instead — see `element_guide`.
- An image with no usable file is not drawn on the storefront.
- Switch image ↔ video in place with `node.kind` (keeps id, name, visibility, box, ratio / fit;
  drops the other settings and words).

## 9. Mistakes the validators reject

- A URL where `media:<id>` is expected; a video URL instead of its id.
- Words for an item / row / column that is not in the Element's list.
- A box key the kind does not accept (e.g. height on `button`, look on `facebook` or `table`),
  or `mar` on a Section / `vmar` on an Element.
- Numbers out of range, wrong types, unknown fields or op names — the whole batch is refused.
- `countdown.at` in seconds or as a date string.
- Unknown icon names; language codes not in `langs`.
- A fifth level of nested groups; more than 400 Elements in a Section; nested links.
- Refused kinds in slots / footer (table above); anything but `tab` directly under `tabs`.
- `node.*` / `words.set` on a system block or on the inside of a placed component.
- Referencing a generated id in the same batch that creates it (supply your own `id` instead).
