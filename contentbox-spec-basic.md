# ContentBox basic spec (chernomoretz1919.bg) — level 1

Reduced rule set for **AI level 1: text and images only**. The output is a fragment of one or more
`.is-section` blocks rendered by the ContentBox runtime inside the Expozy SPA. No `<html>`, `<head>`,
`<body>`, `<style>` or `<script>`.

What level 1 may produce: headings, paragraphs, lists, quotes, images, image + text rows, simple
buttons/links, spacers. Nothing else. For sliders, animations or plugins use level 2; for data
from the Expozy API (news, products, partners…) use level 3 (full spec: `contentbox-spec.md`).

## 1. Section skeleton (always this shape)

```html
<div class="is-section is-section-auto is-box">
    <div class="is-overlay">
        <div class="is-overlay-bg bg-white"></div>
    </div>
    <div class="is-container v3 is-content-1200 px-4 lg:px-8 py-16 lg:py-20">
        <div class="row">
            <div class="column full">…</div>
        </div>
    </div>
</div>
```

- Every top-level block is `.is-section is-section-auto is-box` (content height). No `is-section-N`.
- Background: a Tailwind `bg-*` class on `.is-overlay-bg` (`bg-white`, `bg-[#EEF2F8]`, `bg-[#0C2D6E]`),
  or a photo: `style="background-image:url(…)"` plus `<div class="is-overlay-color" style="opacity:.5"></div>`
  under it when text sits on the photo. Dark section → add `is-light-text` on the section.
- Container: `is-container v3 is-content-1200` (narrow text: `is-content-800`). Keep `v3`.
- Columns: `full`, `half`, `third`, `two-third`. Two columns max per row. Image column + text column
  is the standard "media + text" row; put the image first in the DOM.

## 2. Text

- `<h1>` only in the hero. Section titles `<h2>`, subtitles `<h3>`. Paragraphs `<p>`, lists `<ul>/<ol>`,
  quotes `<blockquote>`.
- Sizes with ContentBox classes, not Tailwind `text-*`: `size-16` body, `size-18`/`size-21` lead,
  `size-28`/`size-32` h3, `size-42`/`size-48` h2, `size-60` hero h1. Line height `leading-14` (body) /
  `leading-11` (headings). Letter spacing on headings `tracking-100`.
- Headings: `font-display uppercase tracking-wide text-[#0C2D6E]` (white on dark sections).
- Body colour `text-slate-600`, muted `text-slate-500`. Alignment `center|left|right`.
- Eyebrow label above a title: `<p class="size-12 uppercase tracking-200 font-bold text-[#153C8C]">Клуб</p>`.
- Bulgarian copy. No lorem ipsum. Keep the client's wording; fix only obvious typos.

## 3. Images

- `<img src="…" alt="…">` with a meaningful `alt`. Uploaded images arrive as absolute `https://r2.expozy.com/…`
  URLs; site assets live in `/static/images/<file>`. Never invent URLs, never use external stock URLs.
- In a column: `class="w-full h-auto"`. Fixed crop: `class="w-full aspect-[4/3] object-cover"`
  (or `aspect-video`). Rounded: `rounded-lg`. Logos: `max-h-20 w-auto`.
- Grid of photos: inside one column, `<div class="grid grid-cols-2 md:grid-cols-3 gap-4">` with plain `<img>`.
- Clickable to full size: `<a class="is-lightbox" href="big.jpg"><img src="big.jpg" alt=""></a>`.
- Caption: `<p class="size-13 text-slate-500 mt-2">…</p>` under the image.

## 4. Buttons & links

```html
<a href="/bg/novini" @click.prevent="href('/novini')"
   class="inline-block no-underline cursor-pointer py-3 px-8 size-16 leading-14 font-display uppercase tracking-wider
          bg-[#0C2D6E] text-white hover:bg-[#153C8C] transition-all">Виж новините</a>
```
- Internal link: `href="/bg/<slug>"` + `@click.prevent="href('/<slug>')"`. External: `target="_blank" rel="noopener"`.
- Outline variant: `border-2 border-[#0C2D6E] text-[#0C2D6E] hover:bg-[#0C2D6E] hover:text-white bg-transparent`.
- Text link inside a paragraph: `<a href="…" class="text-[#153C8C] underline">`.

## 5. Spacing

- Section padding `py-16 lg:py-20` on the container (hero `py-24 lg:py-32`).
- Between elements inside a column: `mt-4` / `mt-6` / `mt-8`, or `<div class="spacer height-40"></div>` (20…300, step 20).
- Column gap is built in. Do not add margins on `.row` or `.column`.

## 6. Not allowed at level 1

- No `data-cb-type` plugins (sliders, accordion, tabs, countdown, forms, maps, video, charts…).
- No animation classes (`animate-*`, `reveal*`, `is-animated`, `delay-*`, `data-fx-*`, `data-bottom-top`).
- No Alpine (`x-data`, `x-for`, `x-show`, `apiData`, `keyName`) and no `lock` sections.
- No `<style>`, `<script>`, `<iframe>`, `<video>`, `<svg>`; no inline `style` except `background-image`,
  `opacity` on `.is-overlay-color` and `width` on a column.
- No Tailwind `size-N` for dimensions (reserved for font size); use `w-*`/`h-*`.
- No new colours: only navy `#0C2D6E`, blue `#153C8C`, light blue `#8FB0E8`, ice `#EEF2F8`, deep `#071E4A`, white, slate greys.

## 7. Patterns

**Hero (title + intro on a photo)**
```html
<div class="is-section is-section-auto is-box is-light-text">
    <div class="is-overlay">
        <div class="is-overlay-bg" style="background-image:url(/static/images/hero-team.jpg)"></div>
        <div class="is-overlay-color" style="background:#071E4A; opacity:.7"></div>
    </div>
    <div class="is-container v3 is-content-1200 px-4 lg:px-8 py-24 lg:py-32">
        <div class="row"><div class="column full">
            <p class="size-12 uppercase tracking-200 font-bold text-[#8FB0E8]">Клуб</p>
            <h1 class="size-60 leading-11 font-display uppercase tracking-wide">Заглавие</h1>
            <p class="size-21 leading-14 mt-4 text-[#8FB0E8]">Кратко въведение.</p>
        </div></div>
    </div>
</div>
```

**Text + image row**
```html
<div class="row">
    <div class="column half"><img src="…" alt="…" class="w-full aspect-[4/3] object-cover"></div>
    <div class="column half">
        <h2 class="size-42 leading-11 font-display uppercase tracking-wide text-[#0C2D6E]">Заглавие</h2>
        <p class="size-16 leading-14 mt-4 text-slate-600">Текст.</p>
    </div>
</div>
```

**Plain text page**: one section, `is-content-800`, `column full`, h2 + paragraphs + lists.

## 8. Checklist

1. Fragment only; every top-level block is `.is-section is-section-auto is-box`.
2. `is-container v3 is-content-N` → `.row` → `.column`, max two columns.
3. Only text, images, lists, quotes, buttons, spacers. Nothing from section 6.
4. ContentBox `size-N`/`leading-NN`/`tracking-N` for type; Tailwind for colour and spacing.
5. Bulgarian copy, `alt` on every image, `href` + `@click.prevent="href()"` on internal links.
