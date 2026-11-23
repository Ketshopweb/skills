# Ai section — a Section you write as one TSX file

An **Ai section** is a Section of type `ai` whose content is a single TSX source (`Element.tsx` in Agent Files). The server checks it, compiles it and attaches the result to the page **draft**. The same mechanism also exists as an **Element AI**: one node of kind `ai` inside a `custom` Section (same source rules, addressed with `nodeId` instead of `sectionId`).

This file is the map. **The full rule text, every limit and the verified examples come from `element_guide`** (no arguments) — call it once before your first `element_check` / `element_write`. It is long; the "Source rules" part is in English.

## When to use it

| Need | Use |
|---|---|
| Text, picture, button, video, FAQ, tabs, slide, table, product highlight … that the Element catalog already has | System Elements in a `custom` Section (`page_apply` ops — see `guide`). The shop owner can edit every part on the canvas. |
| A layout the catalog cannot express: hero, custom grid, cards built from live products, a search box, animation, 3D | Ai section |
| The first picture on the page (hero) | Ai section with a real `<img>` — the system Image Element is always lazy-loaded |
| A custom layout, but the shop must still edit the text / picture / button on the canvas | Ai section + **slots** (below) |

Prefer system Elements when they can do the job. Never rebuild in TSX something the catalog already offers.

## Workflow

1. **Place it.** `page_apply` with `{"op":"section.add","type":"ai","id":"promo"}` (an empty frame appears in the editor), or an Element AI: `{"op":"node.add","sectionId":"hero","kind":"ai","id":"feat"}`.
2. **Read the style.** `design_read` → the table of shop classes and their current values. Optional: `styles_read` for shared classes.
3. **Iterate without saving.** `element_check({ source })` → `{ ok: true, tier, … }` or `problems [{ rule, message, line, col }]`. Needs no page. Repeat until it passes.
4. **Save.** `element_write({ pageId, sectionId | nodeId, source })` → checks, compiles, stores, and points the **current draft** at the new build. Returns `seq`, `pendingChanges`, `previewUrl`, and — when the source has a `meta` — `meta` + `settings`. If the source fails, you get `problems` and the draft is untouched. The same source again = `unchanged: true`.
5. **Set values.** `element_settings({ pageId, sectionId | nodeId, key, value, lang? })` changes one setting declared in `meta` without a new build (`value: null` = back to the default).
6. **Look.** Open `previewUrl` (or call `page_preview`). It shows the whole site as draft on the real storefront.
7. **Publishing is separate** — a person presses "update website", or you use the publish tools with their 2-step confirm (see `site.md`). Never say the site is live after `element_write`.

Edit an existing one: `element_read({ pageId, sectionId | nodeId })` → `source`, `name`, `tier`, `hash`, plus `meta` and the settings in effect (`value`, `isDefault`, per-language values). `hasBuild: false` = still empty. Clear the content: `page_apply` `{"op":"ai.build","id":"promo","build":null}` (you cannot set a build directly — only `element_write` does).

`lockVersion` on `element_write` is optional and ignored: the build goes onto the current draft, so other editors keep their work.

## What the file may contain

- One file, **≤ 32 KB**. Exactly one `export default function <Capitalized>()`; the function name is the build name.
- The outermost tag is one `<section>` or `<div>`. The outer box (background, padding, content width, per-screen values) belongs to the Section and is set outside the source. Design the inside to stretch: mobile first, then `md:` (≥ 768px) and `lg:` (≥ 1024px).
- Imports: `import { useState } from 'react'` and `import { Data, Setting, defineElement, KetCode } from '@ket/code'` (only the names you use). Nothing else — no other package, no `window` / `document` / `fetch` / storage.
- Plain HTML tags from an allowlist (the list is in `element_guide`). No `script`, `iframe`, `style`, `link`; no `style` attribute, `ref`, `{...spread}`, `dangerouslySetInnerHTML`, `srcSet`.
- `className` is a **plain literal string** of standard Tailwind CSS v4 classes. No template strings, expressions or `clsx`. An unknown class is a problem. No `url(...)`, no arbitrary properties / variants (`[mask:…]`, `[&_p]:…`).
- `href` is a literal `#anchor` or a path inside the shop — but see "Links" below.

### Shop design tokens — use them instead of fixed values

Classes that follow the shop's site style, so the Element changes when the shop changes its style (no rebuild):

- Colours `bg-shop-<id>` · `text-shop-<id>` · `border-shop-<id>` with ids `primary` `accent` `text` `muted` `bg` `surface` `line` `price` `on-primary`, and the dark set `dark` `dark-surface` `on-dark` `on-dark-muted` `dark-line` `dark-primary` `dark-price`.
- Corners `rounded-shop-sm|md|lg` · fonts `font-shop-head|body` · sizes `text-shop-h1|h2|body|strong|small` · heading weight `font-shop-heading`.

A fixed brand colour / radius / font (`bg-[#B12629]`, `text-red-600`, `bg-white`, `rounded-xl`, `font-[Prompt]`) passes but returns a `warnings` entry (rule `hardcoded-style`) with a `suggest` class. Fix it; a fixed hex is fine only for colours that are not the shop's brand. To change the look of the whole site, use `design_ops` — do not edit every Element.

### Settings the shop owner can edit (`meta`)

`export const meta = defineElement({ name, form?, settings, data? })` — data, not code (literals only). Each setting becomes a field of the Element's "Settings" window **and** a prop of the component with the same name.

- `Setting.text` / `Setting.textarea` (`i18n: true` = a value per language) · `Setting.number` · `Setting.toggle` · `Setting.select` (2–24 options).
- `Setting.image` — no default. The prop is the **whole** `src` of an `<img>`; empty = `''`, so write `{cover && <img src={cover} … />}`.
- `Setting.link` — the prop is the **whole** `href` of an `<a>`.
- `Setting.richText` — placed only with `<KetCode.RichText html={detail} />`; `i18n` allowed.
- `Setting.products` — a product query the shop can change (below) · `Setting.model3d` — a `.glb` file for `<KetCode.Scene3D>`.

Limits: `name` ≤ 120 chars · ≤ 24 settings · ≤ 4 data keys · names `^[a-z][A-Za-z0-9]{0,31}$`. Not available: a colour setting, a single-product or category picker — do not declare them. A wrong option or value is a problem (rule `meta`), never silently clamped. Props are plain names only (no defaults, renames or `...rest`).

### Product data

- **Never hardcode shop data** (names, prices, discounts, images, product links, SKUs, stock) — not even copied from `products_query`. Declare it in `meta` and receive `items` as a prop (one product = `limit: 1`): `settings: { source: Setting.products({ label: '…', default: { from: 'latest', sort: 'auto', limit: 8 } }) }, data: (s) => ({ items: Data.products(s.source) })`
- Query: `from` = `latest | cat | tag | brand | ids | random | promo` · `sort` = `auto az za low high best rec shop` · `limit` 1–100. Find ids with `catalog_lists` / `products_pool`, test the query with `products_query` (same result as the storefront), then put it in the `default`. Change it later with `element_settings`.
- Each item is a product card (`id name url image priceText inStock …` — the exact shape `products_query` returns). **Pass values through, never build them**: `<a href={p.url}>`, `<img src={p.image} alt={p.name} />`.
- Every list item needs `key={p.id}`. Inside a list `.map`, `.length`, `.slice`, `&&`, ternary and index access work; `.filter`, `.sort` and helper components do not — ask for what you need with `sort` / `limit`.
- A page may hold **≤ 32 distinct product queries**; over that, the save is rejected with the count. Reuse identical queries.
- The footer and the blog sidebar take no product data at all.
- The rules describe product data only. For category lists use the system `prodcats` / `category` Elements.

### Links to the shop's own pages

Do not hard-code system-page paths (`/products`, `/cart`, `/member/login`, `/`): a literal always points at the main-language page, and the URLs of the product list, product detail, blog list, blog post and blog tag pages are set per shop and per language. Take the link from a `Setting.link` prop or from data (`p.url`). `#anchor`, `?query` and the shop's own custom pages (`/about`) are fine as literals. A literal system-page href returns `linkWarnings` (rule `internal-link`) — a warning, not a block.

## Two paths, as you experience them

| | Fast path | Worker path |
|---|---|---|
| Result `tier` | `static` (no JavaScript on the page) or `native` (`useState` open/close) | `worker` |
| When | The source follows every fast-path rule | Tried automatically when the fast path refuses the source |
| Answer | `{ ok: true, tier }` | `{ ok: true, tier: "worker", workerWarnings, workerNote }` — not blocking |
| Cost | none | about 15 KB of JavaScript per page; interactive ~0.1–0.3 s after load |
| Good for | text, links, product lists, simple open/close | form fields and typing, reading the event, `useEffect` / timers, product search while typing (`Data.useProducts`), animation (`<KetCode.Motion>`), 3D (`<KetCode.Scene3D>`) |

- **Write for the fast path whenever you can.** If you get `tier: "worker"` without meaning to, the first line of `workerWarnings` says which fast-path rule was missed — fix it and check again.
- Fast-path handlers are inline, take **no event** (`onClick={() => setOpen(!open)}`), and may only call a `useState` setter or a function declared in the file. With `useState`, do not `.map` a constant list — write the items out.
- Worker path: import **every** hook from `@ket/code` (not `react`). No real DOM, no network, a `ref` cannot measure or focus. `className` must still be a literal — switch whole elements or use `hidden={…}` instead of computing class names. Attributes outside the allowed list are dropped silently.
- Animation is `<KetCode.Motion>` / `<KetCode.AnimatePresence>` only (recipes in `element_guide`). If the worker path is unavailable on the server, a fast-path miss stays a problem.

## Slots — system Elements inside your layout

`<KetCode.Slot name="title" />` marks a place that the shop fills and edits **on the canvas** like any Element (style tab, typing, languages, per screen). You own the place; the shop owns what is inside.

- **Slot or Setting?** Slot = text / picture / button the shop wants to edit on the page. `Setting.*` = a value your own markup uses. Never both for the same thing.
- Source form: exactly self-closing, `name` a plain string, no child, no other attribute. Name `^[a-z][a-z0-9-]{0,31}$`, each name once, ≤ 12 per Section. It renders a `<div>` — put layout classes on the **parent**.
- **Nothing syncs by itself — do both sides together:** (1) `element_write` the source; (2) create one top-level group with the same `slot` name and seed it: `{"op":"node.add","sectionId":"promo","kind":"group","slot":"title","id":"title-slot"}`, then `{"op":"node.add","sectionId":"promo","parentId":"title-slot","kind":"text","id":"title-text"}`, then `{"op":"words.set","sectionId":"promo","patch":{"title-text.text":"…"}}`. Every top-level node of an Ai section must be a slot.
- `slotWarnings` (not blocking) report a name with no group (empty place on the page) or a group whose name left the source (not drawn; removable with `node.remove`).
- While its name is in the source a slot group cannot be removed, moved or duplicated. Move it by changing the source.
- **Not allowed inside a slot:** a shared component (`use`), an Element AI (`ai`), another slot, and the Elements `product`, `blog`, `flashsale`, `slide`, `countdown`, `marquee`, `compare`.
- Slots exist only in an Ai section — not in a `custom` Section and not in an Element AI (there the place stays empty, with a warning).
- On the worker path a slot is a placeholder owned by the page: your code cannot read, change or listen to what is inside.

## Images

- A picture you bring yourself must be in the media library first (`media_upload_link`, or ask the shop to upload). Reference it as `media:<id>` — a typed URL or guessed path is dropped on the page. Product pictures and `Setting.image` values are used exactly as given.
- **Sizes.** Library images are stored as webp. Ask a smaller copy with `?s=<size>`: `WxH`, `W` or `xH` (1–4096). The image is scaled down to fit, never cropped or enlarged — crop with CSS (`object-cover`, `aspect-[…]`). Guide: thumbnail 200 · card 400 · half width 800 · hero 1600. A file holds at most 16 sizes, so reuse 200 / 400 / 800 / 1600. A size that does not exist yet falls back to the next larger one or the full image, never a 404.
- Preferred way: `media_size({ id, size })` → `ref: "media:<id>?s=<size>"`, then `element_settings` on the `Setting.image`. The prop is still the whole `src`, so the Element **stays on the fast path**. Writing `` src={`${cover}?s=800x600`} `` in the source also works but moves the Element to the worker path. Never add `?s=` to a product picture.
- **Hero (first picture on screen):** a real `<img>`, never a CSS background. `loading="eager"`, `decoding="async"`, `fetchPriority="high"`; at most one such picture per page. `fetchPriority` is dropped silently on the worker path, so the hero must stay on the fast path.
- **Below the first screen:** `loading="lazy"` + `decoding="async"` on every `<img>`.
- **Reserve the space:** put the image in a box that already has its shape (`aspect-[16/9] w-full`) with `h-full w-full object-cover` on the `<img>`, plus `width` / `height`. Keep the `{cover && …}` guard **inside** the box.
- No `srcSet` / `<picture>` / `<source>`: one size per place. Icons and logos = SVG. Never a picture of text.

## Global CSS — shared classes

- `styles_read` → `{ css, lockVersion, changedFromLive, live }`. `styles_write({ css, lockVersion, dryRun? })` replaces the **whole file**; a changed `lockVersion` = conflict (read again, merge, write). Problems come back as `[{ path, line, col, rule, message }]`.
- One form only: `@layer components { .btn-primary { @apply rounded-lg bg-blue-600 px-4 py-2 font-semibold text-white hover:bg-blue-700; } }`
- Name: a–z first, then a–z 0–9 `-`, 2–40 chars; ≤ 200 classes, ≤ 60 utilities per class; Tailwind utility names are taken.
- States and screens are variants inside `@apply` (`hover:`, `md:`). No raw CSS, no other selectors or at-rules, no `url(`; `@apply` cannot use another shared class.
- Usable only in Ai section / Element AI sources: `className="btn-primary"` (also `md:btn-primary`). **Write the CSS before `element_write`** of a source that uses a new class, otherwise it is `unknown-class`.
- Draft only. Deleting a class that Elements still use is allowed; the answer lists where (`stillUsed`).

## Typical rejections

| `rule` | Cause → fix |
|---|---|
| `size` | Over 32 KB → shorten the source or split the content over two Sections |
| `default-export` / `import` / `hook` / `global` | Not exactly one capitalized default function; something outside the allowlist (worker path: a hook imported from `react`) |
| `class-literal` / `unknown-class` / `unsafe-class` | `className` not a plain string; not a Tailwind or shared class of the draft; arbitrary property / `url(` |
| `tag` / `attr` / `spread` / `raw-html` | Tag or attribute not allowed; `{...props}`; `dangerouslySetInnerHTML` |
| `handler*` | Fast-path handler reads the event, writes a property or calls something not allowed |
| `href` / `src` | Built or external URL → literal shop path, `Setting.link` / `Setting.image` prop, or a product field as it is |
| `meta` / `props` | Invalid `meta` (line / column given); a prop name not in `settings` ∪ `data` |
| `rich-text` / `slot` / `motion` / `shader` | Wrong form of `KetCode.RichText` / `Slot` / `Motion`, or a banned 3D name |
| `rate` | Too many compiles (120 per minute per connection) — **not a source error**: wait a minute and resend unchanged |

`line` / `col` start at 1 (0 = no position). Fix all problems, then check again. A "temporarily unavailable" error also means retry, not rewrite. Saves can also be refused by the page itself (locked system page, wrong target kind, over 32 product queries).

## Two verified examples

Hero picture on the fast path (`static`). Give `cover` a sized ref: `media_size` → `element_settings({ key: "cover", value: "media:<id>?s=1600" })`.

```tsx
import { Setting, defineElement } from '@ket/code'

export const meta = defineElement({
  name: 'แบนเนอร์หลัก',
  settings: {
    title: Setting.text({ label: 'หัวข้อ', default: 'โปรโมชันประจำเดือน', col: 6 }),
    cover: Setting.image({ label: 'รูปแบนเนอร์ (ขอขนาด 1600 ด้วย media_size)', col: 6 }),
  },
})

export default function HeroBanner({ title, cover }) {
  return (
    <section className="flex flex-col gap-6">
      <div className="aspect-[16/9] w-full overflow-hidden rounded-shop-lg">
        {cover && (
          <img src={cover} alt={title} width="1600" height="900" loading="eager" fetchPriority="high" decoding="async" className="h-full w-full object-cover" />
        )}
      </div>
      <h1 className="text-shop-h1 font-shop-heading text-shop-text">{title}</h1>
    </section>
  )
}
```

Plain text + link button (`static`):

```tsx
export default function Promo() {
  return (
    <section className="flex flex-col gap-4 rounded-shop-md bg-shop-surface p-6 md:flex-row md:items-center">
      <h2 className="text-2xl font-semibold text-shop-text md:flex-1">โปรโมชันประจำเดือน</h2>
      <a href="#offers" className="rounded-shop-md bg-shop-primary px-4 py-2 text-center text-shop-on-primary">ดูโปรโมชัน</a>
    </section>
  )
}
```

`element_guide` has more: live product grid, every setting kind, search while typing, motion, slots, 3D.
