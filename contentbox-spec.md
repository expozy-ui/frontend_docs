# ContentBox page spec (chernomoretz1919.bg)

Reference for generating page HTML that is rendered by the ContentBox runtime
(`/editor/cb/runtime/contentbox-runtime.*`) inside an Expozy SPA (Alpine.js + Tailwind 3 CDN).
Output is a **fragment** of one or more `.is-section` blocks: no `<html>`, `<head>`, `<body>`,
`<style>` or `<script>`. The site wraps it in `#main.is-wrapper` and runs `cbRuntime.init()`.

## 1. Section skeleton

```html
<div class="is-section is-section-auto is-box">
    <div class="is-overlay">
        <div class="is-overlay-bg bg-white"></div>            <!-- color / bg image; optional -->
        <!-- <div class="is-overlay-color" style="opacity:.4"></div>  dark tint over a bg image -->
    </div>
    <div class="is-container v3 is-content-1200 px-4 lg:px-8 py-16 lg:py-20">
        <div class="row">
            <div class="column half">…</div>
            <div class="column half">…</div>
        </div>
    </div>
</div>
```

- `.is-section` is `display:flex; min-height:100vh; overflow:hidden`. Always add a height class:
  `is-section-auto` (content height, **default choice**) or `is-section-N` with N = 10…100 (= N vh).
- `.is-box` stacks content vertically and centers it. `is-content-top` / `is-content-bottom` change vertical align.
- `.is-overlay` is absolute, behind the content. `.is-overlay-bg` takes a Tailwind `bg-*` class or
  `style="background-image:url(…)"` (cover, 50% 60%). Responsive bg: `data-bg-xs|sm|md|lg="background-position:…"`.
- `.is-container` = `margin:auto; padding:0 20px`. Add `v3` so sizes stay in px (without `v3` the `v2`
  fluid mode scales `size-*` and spacers with viewport). Max width: `is-content-N` (N = 300…2700 step 20)
  or `is-content-none`. Position: `is-content-left` / `is-content-right`.
- Full-bleed section (hero, marquee, banner): `<div class="is-container v3 !max-w-full !px-0 !py-0">`
  then `<div class="row !mx-0 !gap-0"><div class="column !px-0">` and build the layout with Tailwind inside.
- Text colour helpers on section/box: `is-light-text` (white text) / `is-dark-text`.
- Multiple boxes side by side: `.is-section > .is-boxes > .is-box.is-box-6` (1…12 grid). They stack at ≤970px.
- `.lock` on a section marks it as dynamic (Alpine/apiData) and non-editable in the visual editor.

## 2. Grid

- `.row` is flex ≥761px, block (stacked) below. `.column` widths: `full half third fourth fifth sixth`
  `two-third two-fourth two-fifth`, or inline `style="width:40%; flex:0 0 auto"`. Column padding `0 1rem`.
- Tailwind `grid`/`flex` classes may be used **inside** a column instead of nested rows (preferred for complex layouts).
- Visibility: `xs-hidden` (≤760) `sm-hidden` (761–970) `md-hidden` (971–1280) `desktop-hidden` (≥1281).
  Mobile order: `sm-column-reverse`, `md-column-reverse`.

## 3. Styling rules

- **All standard Tailwind 3 utilities are allowed** (CDN build, config in `/assets/js/tailwind.config.js`,
  `important: '#body'`). Arbitrary values (`text-[#0C2D6E]`, `[clip-path:…]`, `[animation-delay:120ms]`) work.
- Tailwind `size-N` is **blocked**. In this project `size-N` = ContentBox font-size in px
  (N = 12…21, 24, 28, 32, 35, 38, 42, 46, 48, 50, 54, 60…400 step 4). Use `w-*`/`h-*` for dimensions.
- ContentBox text utilities: `leading-NN` (NN/10, e.g. `leading-14` = 1.4), `tracking-N` (N/1000 em: 75…500,
  `tracking--50/-75/-100`), `font-light|normal|medium|semibold|bold`, `center|left|right`.
- Vertical space: `<div class="spacer height-N"></div>` (N = 20…300 step 20) or Tailwind `py-*`/`mt-*`.
- Typography preset on a section: `type-system-ui` (system font, h1 ≈ 2.25rem). The site font is
  `Repo Regular`; display headings use `font-display` (Oswald) + `uppercase tracking-wide`.
- Site palette: navy `#0C2D6E`, blue `#153C8C`, light blue `#8FB0E8`, ice `#EEF2F8`, deep `#071E4A`, white.
  Custom utilities: `text-outline`, `text-outline-2`, `cut-corner`, `cut-corner-lg`, `slant-b`, `slant-t`.
- No custom CSS, no `<style>`, no inline `style` except where ContentBox requires it (bg-image, column width, hover vars).
- Images: `/static/images/<file>` or absolute URLs; always `alt`. Icons: Font Awesome 6 (`<i class="fa-solid fa-arrow-right">`).

## 4. Buttons & links

```html
<a href="/bg/novini" @click.prevent="href('/novini')"
   class="transition-all inline-block whitespace-nowrap cursor-pointer no-underline border-2 border-solid border-transparent
          py-3 px-8 size-16 leading-14 rounded-full font-display uppercase tracking-wider bg-[#0C2D6E] text-white hover:bg-[#153C8C]">
   Виж новините</a>
```
- Internal links: `href="/bg/<slug>"` plus `@click.prevent="href('/<slug>')"` (SPA router). External: `target="_blank" rel="noopener"`.
- Outline variant: `border-[#0C2D6E] text-[#0C2D6E] hover:bg-[#0C2D6E] hover:text-white`.
- Lightbox: `<a class="is-lightbox" href="big.jpg"><img src="thumb.jpg"></a>`; group with `data-gallery="g1"`
  (GLightbox is bundled; `class="glightbox" data-title data-description` also works).
- Scroll to section: `<a href="#id">` smooth-scrolls; `.is-arrow-down a` scrolls to the next section.

## 5. Animations

Pick **one** system per element. All respect `prefers-reduced-motion`.

**A. Tailwind (preferred, lightweight)**
- On load: `animate-fade-up | fade-in | slide-in | slide-left | clip-up | tilt-in | grow-x | pop | float | stripes
  | pulse-soft | ken-burns | marquee | shine`; stagger with `[animation-delay:120ms]`.
- On scroll (scroll-driven, runs once on load in old browsers): `reveal` (fade-up) `reveal-clip` `reveal-tilt`
  `reveal-left` `reveal-right` `reveal-line` (grow-x, origin left) `reveal-pop`; siblings get `stagger-1|2|3`.
  `draw-y` (vertical line draws while scrolling), `scroll-progress`.
- Hover: standard Tailwind `group-hover:*`, `hover:scale-*`, `transition-*`, `duration-*`.

**B. ContentBox in-view classes** (runtime adds `is-inview` when the element enters the viewport)
- `is-animated` + effect: `is-fadeIn is-fadeInUp is-fadeInDown is-fadeInLeft is-fadeInRight is-zoomIn is-zoomOut
  is-slideInUp/Down/Left/Right is-flipInX is-flipInY is-bounceIn is-pulse`.
- Delay: `delay-100ms` … `delay-3000ms` (step 100). Add `once` to animate only the first time.
  Siblings inside one container block are auto-staggered by 0.2s when no `delay-*` is set.
- Example: `<h2 class="is-animated is-fadeInUp delay-200ms once size-42">…</h2>`

**C. Scroll effects `data-fx-*`** (advanced; never combine with legacy `data-bottom-top` on the same element)
- Keyframes as % of scroll progress: `data-fx-0="opacity:0; transform:translateY(40px)" data-fx-40="opacity:1; transform:none"`
  plus `data-fx-ease="expoOut"`.
- Presets: `data-fx-parallax="120"`, `data-fx-split="words|chars"`, `data-fx-typewriter`, `data-fx-draw` (SVG),
  `data-fx-tilt`, `data-fx-magnetic`, `data-fx-marquee data-fx-marquee-speed="40"`.
- Section options: `data-fx-scrub="0.25"`, `data-fx-once`, `data-fx-off`. Pinned sections: `class="section-pin" data-fx-length="300"`.

**D. Legacy skrollr** (only when editing older blocks): `data-bottom-top="transform:translateX(130px)"`,
`data-center-top="transform:translateX(0)"`, `data-top-bottom`, `data-50-top`. Disable with `data-skrollrr-off`.

## 6. Sliders

Plugins are mounted by `data-cb-type="<name>"`. Every option is a `data-cb-*` attribute (kebab-case → camelCase).
**Defaults are not applied at runtime: always write every attribute listed.** Save the clean markup below;
the runtime generates arrows/dots itself.

**swiper-slider** (image/video slider, Swiper 11 loaded from CDN)
```html
<div data-cb-type="swiper-slider" data-cb-autoplay="true" data-cb-delay="5000" data-cb-transition-speed="800"
     data-cb-loop="true" data-cb-pause-on-hover="true" data-cb-effect="slide" data-cb-zoom-effect="true" data-cb-zoom-scale="108"
     data-cb-aspect-ratio="16/9" data-cb-slides-per-view="1" data-cb-space-between="0" data-cb-rounded-corners="0"
     data-cb-text-position="position-bottom-left" data-cb-text-max-width="700"
     data-cb-top-overlay-opacity="0.3" data-cb-bottom-overlay-opacity="0.3"
     data-cb-navigation="true" data-cb-navigation-style="transparent" data-cb-navigation-button-size="48"
     data-cb-navigation-arrow-size="16" data-cb-navigation-offset="20" data-cb-navigation-rounded-corners="50"
     data-cb-pagination="true" data-cb-pagination-style="dots">
  <div class="swiper">
    <div class="swiper-wrapper">
      <div class="swiper-slide">
        <div class="slide-image">
          <img src="/static/images/team.jpg" alt="">
          <div class="slide-overlay"></div>
          <div class="slide-caption"><div class="is-subblock edit">
            <h3 class="slide-caption-title">Заглавие</h3><p class="slide-caption-description">Текст</p>
          </div></div>
        </div>
      </div>
      <div class="swiper-slide"><div class="slide-image"><img src="/static/images/fans.jpg" alt=""></div></div>
    </div>
    <div class="swiper-button-prev"></div><div class="swiper-button-next"></div><div class="swiper-pagination"></div>
  </div>
</div>
```
Options: `effect` slide|fade|cube|coverflow|flip · `aspect-ratio` 16/9|4/3|1/1|3/4|auto · `text-position` position-{top|center|bottom}-{left|center|right}
· `navigation-style` transparent|solid|arrow · `pagination-style` dots|rounded-rectangle|lines.
Video slide: `<video autoplay loop muted playsinline data-src-desktop="…" data-src-mobile="…"><source src="…" type="video/mp4"></video>`.

**media-slider** (cards carousel, N per view, no external lib)
```html
<div data-cb-type="media-slider" data-cb-items-per-slide="3" data-cb-gap="16" data-cb-aspect-ratio="4/3"
     data-cb-accent-color="#0C2D6E" data-cb-border-color="#e5e7eb" data-cb-rounded="true"
     data-cb-autoplay="false" data-cb-autoplay-speed="5000" data-cb-loop="true" data-cb-pause-on-hover="true"
     data-cb-show-arrows="true" data-cb-show-dots="true" data-cb-show-counter="false">
  <div class="media-slider-track">
    <div class="slider-item">
      <div class="item-media"><img src="/static/images/a.jpg" alt=""></div>
      <div class="item-content"><h3 class="item-title">Заглавие</h3><p class="item-description">Текст</p>
        <a href="/bg/novini" class="item-link">Още →</a></div>
    </div>
    <!-- more .slider-item -->
  </div>
</div>
```

**text-slider** (testimonials/quotes, Swiper)
```html
<div data-cb-type="text-slider" data-cb-autoplay="true" data-cb-delay="4000" data-cb-transition-speed="600"
     data-cb-loop="true" data-cb-pause-on-hover="true" data-cb-effect="slide" data-cb-slides-per-view="1"
     data-cb-space-between="24" data-cb-text-align="center" data-cb-navigation="true" data-cb-pagination="true"
     data-cb-accent-color="#0C2D6E" data-cb-background="transparent" data-cb-bg-color="#ffffff"
     data-cb-border="true" data-cb-border-color="#e5e7eb" data-cb-rounded="12">
  <div class="swiper"><div class="swiper-wrapper">
    <div class="swiper-slide"><div class="text-slide"><div class="is-subblock edit">
      <h3 class="text-slide-title">Заглавие</h3><p class="text-slide-text">Текст</p></div></div></div>
  </div><div class="swiper-button-prev"></div><div class="swiper-button-next"></div><div class="swiper-pagination"></div></div>
</div>
```

**fullscreen-slider** (crossfade hero background; place inside `.is-overlay` of a `is-section-100` section)
```html
<div data-cb-type="fullscreen-slider" data-cb-text-position="position-center" data-cb-autoplay-duration="6000"
     data-cb-transition-speed="1200" data-cb-enable-keyboard="true" data-cb-pause-on-hover="true"
     data-cb-show-counter="false" data-cb-show-nav-buttons="true" data-cb-show-pagination="true"
     data-cb-show-keyboard-hint="false" data-cb-zoom-effect="true" data-cb-zoom-scale="108" data-cb-overlay-opacity="30">
  <div class="media-overlay"></div>
  <div class="media-slide active"><div class="media-slide-inner"><img src="/static/images/1.jpg" alt=""></div>
    <div class="item-content position-center is-container"><div class="is-subblock edit"><h2>Заглавие</h2><p>Текст</p></div></div>
  </div>
  <div class="media-slide"><div class="media-slide-inner"><img src="/static/images/2.jpg" alt=""></div></div>
</div>
```
It always autoplays; `autoplay-duration` is mandatory.

**Alpine fade slider** (no plugin; used by the homepage hero). Fine for simple cross-fades:
`x-data="{ slide: 0, slides: ['/static/images/a.jpg','/static/images/b.jpg'] }" x-init="setInterval(() => slide = (slide + 1) % slides.length, 6000)"`
with `<template x-for="(src, i) in slides"><img :src="src" x-show="slide === i" x-transition.opacity.duration.1000ms class="absolute inset-0 w-full h-full object-cover"></template>`.

## 7. Other plugins (same `data-cb-type` + `data-cb-*` rule)

`accordion faq tabs timeline process-steps pricing-table testimonials team-members card-list flip-card callout-box
simple-stats animated-stats progress-bars click-counter countdown-timer star-rating typewriter marquee marquee-wall
logo-loop before-after-slider media-grid floating-showcase browser-mockup hero-animation hero-background blobs-bg
aurora-glow particle-constellation vector-force pendulum video-background video-embed youtube-embed audio-player
map-embed chart code code-view contact-form form social-share section-divider table-of-contents nav-menu
sticky-category-nav more-info cta-buttons`. Prefer plain Tailwind markup for simple cases (cards, stats, CTA);
use a plugin only when it adds real behaviour (slider, accordion, tabs, countdown, before/after, canvas effects).

## 8. Dynamic data (Expozy)

Sections that load API data use `apiData="get.<endpoint>" keyName="<key>"` on the section and Alpine
`<template x-for="item in data.<key>.result">`. Mark such sections with `lock`. Pure content pages need none of this.

## 9. Checklist

1. Fragment only; each top-level block is `.is-section` with a height class.
2. `is-container v3` + `is-content-N` (or full-bleed variant) → `.row` → `.column`.
3. Tailwind for layout/colour/spacing; `size-N`/`leading-NN`/`tracking-N` for ContentBox typography; no `<style>`.
4. Animations: Tailwind `animate-*`/`reveal*` or `is-animated is-*` (+ `delay-*`, `once`); `data-fx-*` for scroll effects.
5. Sliders via `data-cb-type` with **all** `data-cb-*` attributes and clean slide markup.
6. Bulgarian copy, `alt` on images, `href` + `@click.prevent="href()"` on internal links.
