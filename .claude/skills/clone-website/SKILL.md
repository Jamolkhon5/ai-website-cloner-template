---
name: clone-website
description: Reverse-engineer and clone one or more websites in one shot — extracts assets, CSS, content AND interactive behaviour section-by-section, then dispatches parallel builder agents that implement WORKING React (real modals, toasts, tabs, forms, theme) — not dead static markup. Use this whenever the user wants to clone, replicate, rebuild, reverse-engineer, or copy any website. Also triggers on phrases like "make a copy of this site", "rebuild this page", "pixel-perfect clone", "make the buttons work". Provide one or more target URLs as arguments.
argument-hint: "<url1> [<url2> ...]"
user-invocable: true
---

# Clone Website

You are about to reverse-engineer and rebuild **$ARGUMENTS** as pixel-perfect clones.

When multiple URLs are provided, process them independently and in parallel where possible, while keeping each site's extraction artifacts isolated in dedicated folders (for example, `docs/research/<hostname>/`).

This is not a two-phase process (inspect then build). You are a **foreman walking the job site** — as you inspect each section of the page, you write a detailed specification to a file, then hand that file to a specialist builder agent with everything they need. Extraction and construction happen in parallel, but extraction is meticulous and produces auditable artifacts.

## Scope Defaults

The target is the application `$ARGUMENTS` resolves to. Clone the **whole reachable application** — the landing/entry URL AND every route reachable from it (including routes behind a login wall). Do NOT stop at the single entry URL, and do NOT stop at a login page. Unless the user specifies otherwise, use these defaults:

- **Fidelity level:** Pixel-perfect — exact match in colors, spacing, typography, animations
- **In scope:** Visual layout and styling, component structure and **all interactive states** (modals, toasts, loading/empty/error, hover/active/disabled, theme variants), responsive design, **every reachable route**, mock data for demo purposes
- **Authentication: IN SCOPE.** If a route is gated by a login / 2FA wall, authenticate so you can reach the real authenticated surface — that surface IS the product. To get credentials, call the `operator-bridge` MCP tool `request_credentials` (and `request_code` for a 2FA/verification code). These tools block until a human operator replies, then return the secret. Log in with the reply and continue. If a tool returns `NO_OPERATOR_RESPONSE` (no operator available), degrade gracefully to cloning only the public/pre-auth surface — never guess or brute-force credentials.
- **Out of scope:** Real backend / database, real-time features, SEO optimization, accessibility audit, and any **elevated / SuperAdmin-only** area (clone the normal authenticated user surface only).
- **Customization:** None — pure emulation

If the user provides additional instructions (specific fidelity level, customizations, extra context), honor those over the defaults.

## Pre-Flight

1. **Browser automation is required.** Check for available browser MCP tools (Chrome MCP, Playwright MCP, Browserbase MCP, Puppeteer MCP, etc.). Use whichever is available — if multiple exist, prefer Chrome MCP. If none are detected, ask the user which browser tool they have and how to connect it. This skill cannot work without browser automation.
2. Parse `$ARGUMENTS` as one or more URLs. Normalize and validate each URL; if any are invalid, ask the user to correct them before proceeding. For each valid URL, verify it is accessible via your browser MCP tool.
3. **Authenticate FIRST if the entry page is a login wall.** Navigate to the entry URL and detect a login form (a password field, "Sign in" submit, or SSO buttons). If present, the real target is *behind* it: call the `operator-bridge` MCP `request_credentials` tool, wait for the operator's reply, fill the form and submit (call `request_code` if a 2FA/verification prompt appears), and confirm you land on an authenticated page before doing recon. The authenticated surface is the product — recon run only on the login page captures ~nothing. If `request_credentials` returns `NO_OPERATOR_RESPONSE`, proceed with the public surface only.
4. Verify the base project builds: `npm run build`. The Next.js + shadcn/ui + Tailwind v4 scaffold should already be in place. If not, tell the user to set it up first.
5. Create the output directories if they don't exist: `docs/research/`, `docs/research/components/`, `docs/design-references/`, `scripts/`. For multiple clones, also prepare per-site folders like `docs/research/<hostname>/` and `docs/design-references/<hostname>/`.
6. When working with multiple sites in one command, optionally confirm whether to run them in parallel (recommended, if resources allow) or sequentially to avoid overload.

## Guiding Principles

These are the truths that separate a successful clone from a "close enough" mess. Internalize them — they should inform every decision you make.

### 1. Completeness Beats Speed

Every builder agent must receive **everything** it needs to do its job perfectly: screenshot, exact CSS values, downloaded assets with local paths, real text content, component structure. If a builder has to guess anything — a color, a font size, a padding value — you have failed at extraction. Take the extra minute to extract one more property rather than shipping an incomplete brief.

### 2. Small Tasks, Perfect Results

When an agent gets "build the entire features section," it glosses over details — it approximates spacing, guesses font sizes, and produces something "close enough" but clearly wrong. When it gets a single focused component with exact CSS values, it nails it every time.

Look at each section and judge its complexity. A simple banner with a heading and a button? One agent. A complex section with 3 different card variants, each with unique hover states and internal layouts? One agent per card variant plus one for the section wrapper. When in doubt, make it smaller.

**Complexity budget rule:** If a builder prompt exceeds ~150 lines of spec content, the section is too complex for one agent. Break it into smaller pieces. This is a mechanical check — don't override it with "but it's all related."

### 3. Real Content, Real Assets

Extract the actual text, images, videos, and SVGs from the live site. This is a clone, not a mockup. Use `element.textContent`, download every `<img>` and `<video>`, extract inline `<svg>` elements as React components. The only time you generate content is when something is clearly server-generated and unique per session.

**Layered assets matter.** A section that looks like one image is often multiple layers — a background watercolor/gradient, a foreground UI mockup PNG, an overlay icon. Inspect each container's full DOM tree and enumerate ALL `<img>` elements and background images within it, including absolutely-positioned overlays. Missing an overlay image makes the clone look empty even if the background is correct.

### 4. Foundation First

Nothing can be built until the foundation exists: global CSS with the target site's design tokens (colors, fonts, spacing), TypeScript types for the content structures, and global assets (fonts, favicons). This is sequential and non-negotiable. Everything after this can be parallel.

### 5. Extract How It Looks AND How It Behaves

A website is not a screenshot — it's a living thing. Elements move, change, appear, and disappear in response to scrolling, hovering, clicking, resizing, and time. If you only extract the static CSS of each element, your clone will look right in a screenshot but feel dead when someone actually uses it.

For every element, extract its **appearance** (exact computed CSS via `getComputedStyle()`) AND its **behavior** (what changes, what triggers the change, and how the transition happens). Not "it looks like 16px" — extract the actual computed value. Not "the nav changes on scroll" — document the exact trigger (scroll position, IntersectionObserver threshold, viewport intersection), the before and after states (both sets of CSS values), and the transition (duration, easing, CSS transition vs. JS-driven vs. CSS `animation-timeline`).

Examples of behaviors to watch for — these are illustrative, not exhaustive. The page may do things not on this list, and you must catch those too:
- A navbar that shrinks, changes background, or gains a shadow after scrolling past a threshold
- Elements that animate into view when they enter the viewport (fade-up, slide-in, stagger delays)
- Sections that snap into place on scroll (`scroll-snap-type`)
- Parallax layers that move at different rates than the scroll
- Hover states that animate (not just change — the transition duration and easing matter)
- Dropdowns, modals, accordions with enter/exit animations
- Scroll-driven progress indicators or opacity transitions
- Auto-playing carousels or cycling content
- Dark-to-light (or any theme) transitions between page sections
- **Tabbed/pill content that cycles** — buttons that switch visible card sets with transitions
- **Scroll-driven tab/accordion switching** — sidebars where the active item auto-changes as content scrolls past (IntersectionObserver, NOT click handlers)
- **Smooth scroll libraries** (Lenis, Locomotive Scroll) — check for `.lenis` class or scroll container wrappers

### 6. Identify the Interaction Model Before Building

This is the single most expensive mistake in cloning: building a click-based UI when the original is scroll-driven, or vice versa. Before writing any builder prompt for an interactive section, you must definitively answer: **Is this section driven by clicks, scrolls, hovers, time, or some combination?**

How to determine this:
1. **Don't click first.** Scroll through the section slowly and observe if things change on their own as you scroll.
2. If they do, it's scroll-driven. Extract the mechanism: `IntersectionObserver`, `scroll-snap`, `position: sticky`, `animation-timeline`, or JS scroll listeners.
3. If nothing changes on scroll, THEN click/hover to test for click/hover-driven interactivity.
4. Document the interaction model explicitly in the component spec: "INTERACTION MODEL: scroll-driven with IntersectionObserver" or "INTERACTION MODEL: click-to-switch with opacity transition."

A section with a sticky sidebar and scrolling content panels is fundamentally different from a tabbed interface where clicking switches content. Getting this wrong means a complete rewrite, not a CSS tweak.

### 7. Extract Every State, Not Just the Default

Many components have multiple visual states — a tab bar shows different cards per tab, a header looks different at scroll position 0 vs 100, a card has hover effects. You must extract ALL states, not just whatever is visible on page load.

For tabbed/stateful content:
- Click each tab/button via browser MCP
- Extract the content, images, and card data for EACH state
- Record which content belongs to which state
- Note the transition animation between states (opacity, slide, fade, etc.)

For scroll-dependent elements:
- Capture computed styles at scroll position 0 (initial state)
- Scroll past the trigger threshold and capture computed styles again (scrolled state)
- Diff the two to identify exactly which CSS properties change
- Record the transition CSS (duration, easing, properties)
- Record the exact trigger threshold (scroll position in px, or viewport intersection ratio)

For **async data states** (these are invisible at page load and are the most commonly missed — a real data app spends most of its life in one of these):
- **Loading:** throttle the network (browser MCP slow-3G / offline-then-online) or reload and capture the spinner / skeleton frame BEFORE data arrives.
- **Empty:** reach a list with zero rows (filter to nothing, or open a fresh/empty resource) and capture the empty-state copy + illustration.
- **Error:** force a request to fail (block the endpoint / go offline mid-load) and capture the error toast + any fallback UI.
- Record the exact copy for each — empty/error strings are real product copy and cannot be invented.

### 8. Spec Files Are the Source of Truth

Every component gets a specification file in `docs/research/components/` BEFORE any builder is dispatched. This file is the contract between your extraction work and the builder agent. The builder receives the spec file contents inline in its prompt — the file also persists as an auditable artifact that the user (or you) can review if something looks wrong.

The spec file is not optional. It is not a nice-to-have. If you dispatch a builder without first writing a spec file, you are shipping incomplete instructions based on whatever you can remember from a browser MCP session, and the builder will guess to fill gaps.

### 9. Build Must Always Compile

Every builder agent must verify `npx tsc --noEmit` passes before finishing. After merging worktrees, you verify `npm run build` passes. A broken build is never acceptable, even temporarily.

### 10. A Clone That Doesn't WORK Is a Failed Clone

This is the principle that separates a screenshot from a clone. **Pixel-perfect appearance with dead buttons is a FAILURE.** If a user clicks "Send SMS" and no modal opens, clicks a tab and nothing switches, submits an empty form and no validation toast fires, or toggles the theme and nothing changes — the clone is not done, no matter how perfect it looks frozen.

Capturing a modal's DOM/copy into a spec file is **not** the goal — it is the input. The goal is that the cloned button, when clicked, **actually opens that modal**. Every interactive surface you captured in the Interaction Sweep MUST be **implemented as working React behaviour**, not rendered as static markup. The concrete rules:

- **Any page with an interactive surface is a client component** — it starts with `"use client"` and holds real state (`useState`/`useReducer`). A static server component that renders a button it never wires is the #1 clone defect. The default-export page (or the interactive sub-component) must be `"use client"`.
- **Triggers are wired, not decorative.** Every `<button>`/menu/tab/row-action you captured gets a real `onClick` (or `onChange`) that drives state — opening a modal, switching a tab, firing a toast, mutating a mock list.
- **Behaviour runs against mock data, not a real backend.** Seed an in-file mock array/object so lists render rows, pagination slices, empty-filters show the empty state, and a `loading` flag can show the captured spinner. NEVER `fetch()` the original backend (it won't be there, breaks `output:export`, and leaks the original's API). Mock it client-side.
- **Verbatim captured copy is the product.** Toast strings, empty-state copy, confirm "Are you sure…" text, validation messages — use the EXACT strings you captured. Do not invent or paraphrase.

The dedicated **Phase 3.5 (Behavior Implementation)** below makes this mechanical, and the **Phase 5.5 functional gate** proves it by re-driving the clone's OWN buttons.

## Phase 1: Reconnaissance

Navigate to the target URL with browser MCP. **If it is a login wall, authenticate first** (Pre-Flight step 3) — recon must run on the real authenticated app, not the login page.

### Route Enumeration (do this FIRST, before per-page recon)

A real app is many routes, not one page. Before any per-page work, enumerate every reachable route and recon **each** one. Skipping this is the single biggest miss — a clone of only the entry route reproduces a fraction of the product.

1. **Discover routes** from the authenticated app: read every sidebar/nav `<a href>`, header menu, dashboard tiles, in-app router links, footer links, and any `sitemap.xml`. Also probe obvious siblings of what you see (e.g. if `/campaigns` exists, check `/campaigns/new`, a detail route, a trash/archive route).
2. **Build a ROUTE LIST** — the deduplicated set of in-app paths. Cap at **25 routes** (if more, prioritize: every distinct top-level feature first, then sub-routes). For **dynamic routes** (`/items/:id`), clone ONE representative instance per pattern (open a real record, note the path is a template).
3. **Recon EACH route** — run the Screenshots + Global Extraction (first route only for global tokens) + Mandatory Interaction Sweep (every route) below, once per route. Namespace artifacts per route, e.g. `docs/research/<route-slug>/`.
4. Record the full route list + which are public vs. authenticated + the nav structure in `docs/research/ROUTE_MAP.md`. This is the breadth blueprint; `PAGE_TOPOLOGY.md` (below) is written per route.

The Screenshots / Global Extraction / Interaction Sweep / Page Topology steps below now run **per route in the ROUTE LIST** (global tokens — fonts/colors/favicons — are extracted once from the first route and reused).

### Screenshots
- Take **full-page screenshots** at desktop (1440px) and mobile (390px) viewports
- Save to `docs/design-references/` with descriptive names
- These are your master reference — builders will receive section-specific crops/screenshots later

### Global Extraction
Extract these from the page before doing anything else:

**Fonts** — Inspect `<link>` tags for Google Fonts or self-hosted fonts. Check computed `font-family` on key elements (headings, body, code, labels). Document every family, weight, and style actually used. Configure them in `src/app/layout.tsx` using `next/font/google` or `next/font/local`.

**Colors** — Extract the site's color palette from computed styles across the page. Update `src/app/globals.css` with the target's actual colors in the `:root` and `.dark` CSS variable blocks. Map them to shadcn's token names (background, foreground, primary, muted, etc.) where they fit. Add custom properties for colors that don't map to shadcn tokens.

**Favicons & Meta** — Download favicons, apple-touch-icons, OG images, webmanifest to `public/seo/`. Update `layout.tsx` metadata.

**Global UI patterns** — Identify any site-wide CSS or JS: custom scrollbar hiding, scroll-snap on the page container, global keyframe animations, backdrop filters, gradients used as overlays, **smooth scroll libraries** (Lenis, Locomotive Scroll — check for `.lenis`, `.locomotive-scroll`, or custom scroll container classes). Add these to `globals.css` and note any libraries that need to be installed.

### Mandatory Interaction Sweep

This is a dedicated pass AFTER screenshots and BEFORE anything else. Its purpose is to discover every behavior on the page — many of which are invisible in a static screenshot.

**Scroll sweep:** Scroll the page slowly from top to bottom via browser MCP. At each section, pause and observe:
- Does the header change appearance? Record the scroll position where it triggers.
- Do elements animate into view? Record which ones and the animation type.
- Does a sidebar or tab indicator auto-switch as you scroll? Record the mechanism.
- Are there scroll-snap points? Record which containers.
- Is there a smooth scroll library active? Check for non-native scroll behavior.

**Click sweep — exhaustive. This is where most fidelity is won or lost.** Drive EVERY interactive surface and record its result. Do not skip a surface because reaching it needs a click — that is exactly what a screenshot misses. Per route, work through this matrix:

- **Every button** — record all 4 visual states (default / hover / active / disabled) for each variant and size, plus the button's label.
- **Every modal / dialog** — click each trigger, OPEN the modal, and capture: full DOM, every form field + its placeholder/label, the validation behaviour (submit empty / invalid → capture the inline error AND the toast), and the success path. Many surfaces are reachable ONLY via a modal — an unopened modal is an unclonable modal.
- **Every toast / alert** — perform each action's success AND failure path and record the toast **copy verbatim**, plus its color, icon, position, and duration. Toast strings are real product copy — they cannot be invented; you must read them.
- **Every confirm / delete / restore dialog** — trigger the destructive action and capture the exact confirmation copy ("Are you sure…"), whether it is a custom danger modal or a native `window.confirm`.
- **Every tab set** — click EACH tab and record each panel's content + the active/inactive styles + the switch animation.
- **Header surfaces** — open the user/profile dropdown (record items + divider), the notifications bell panel (record items + the empty state), and toggle the **theme** (light↔dark): the theme toggle yields a FULL second color variant of the page — capture both.
- **Pagination / row-action menus** — exercise next/prev/limit and any per-row "…" action menus; record the controls and offset behaviour.

For tabs/pills/stateful content: click EACH ONE and record the content that appears for each state (this already-existing rule still applies, now under the fuller matrix above).

**Hover sweep:** Hover over every element that might have hover states:
- Buttons, cards, links, images, nav items
- Record what changes: color, scale, shadow, underline, opacity

**Responsive sweep:** Test at 3 viewport widths via browser MCP:
- Desktop: 1440px
- Tablet: 768px
- Mobile: 390px
- At each width, note which sections change layout (column → stack, sidebar disappears, etc.) and at approximately which breakpoint the change occurs.

Save all findings to `docs/research/BEHAVIORS.md`. This is your behavior bible — reference it when writing every component spec.

### Page Topology
Map out every distinct section of the page from top to bottom. Give each a working name. Document:
- Their visual order
- Which are fixed/sticky overlays vs. flow content
- The overall page layout (scroll container, column structure, z-index layers)
- Dependencies between sections (e.g., a floating nav that overlays everything)
- **The interaction model** of each section (static, click-driven, scroll-driven, time-driven)

Save this as `docs/research/PAGE_TOPOLOGY.md` — it becomes your assembly blueprint.

## Phase 2: Foundation Build

This is sequential. Do it yourself (not delegated to an agent) since it touches many files:

1. **Update fonts** in `layout.tsx` to match the target site's actual fonts
2. **Update globals.css** with the target's color tokens, spacing values, keyframe animations, utility classes, and any **global scroll behaviors** (Lenis, smooth scroll CSS, scroll-snap on body)
3. **Create TypeScript interfaces** in `src/types/` for the content structures you've observed
4. **Extract SVG icons** — find all inline `<svg>` elements on the page, deduplicate them, and save as named React components in `src/components/icons.tsx`. Name them by visual function (e.g., `SearchIcon`, `ArrowRightIcon`, `LogoIcon`).
5. **Download global assets** — write and run a Node.js script (`scripts/download-assets.mjs`) that downloads all images, videos, and other binary assets from the page to `public/`. Preserve meaningful directory structure.
6. Verify: `npm run build` passes

### Asset Discovery Script Pattern

Use browser MCP to enumerate all assets on the page:

```javascript
// Run this via browser MCP to discover all assets
JSON.stringify({
  images: [...document.querySelectorAll('img')].map(img => ({
    src: img.src || img.currentSrc,
    alt: img.alt,
    width: img.naturalWidth,
    height: img.naturalHeight,
    // Include parent info to detect layered compositions
    parentClasses: img.parentElement?.className,
    siblings: img.parentElement ? [...img.parentElement.querySelectorAll('img')].length : 0,
    position: getComputedStyle(img).position,
    zIndex: getComputedStyle(img).zIndex
  })),
  videos: [...document.querySelectorAll('video')].map(v => ({
    src: v.src || v.querySelector('source')?.src,
    poster: v.poster,
    autoplay: v.autoplay,
    loop: v.loop,
    muted: v.muted
  })),
  backgroundImages: [...document.querySelectorAll('*')].filter(el => {
    const bg = getComputedStyle(el).backgroundImage;
    return bg && bg !== 'none';
  }).map(el => ({
    url: getComputedStyle(el).backgroundImage,
    element: el.tagName + '.' + el.className?.split(' ')[0]
  })),
  svgCount: document.querySelectorAll('svg').length,
  fonts: [...new Set([...document.querySelectorAll('*')].slice(0, 200).map(el => getComputedStyle(el).fontFamily))],
  favicons: [...document.querySelectorAll('link[rel*="icon"]')].map(l => ({ href: l.href, sizes: l.sizes?.toString() }))
});
```

Then write a download script that fetches everything to `public/`. Use batched parallel downloads (4 at a time) with proper error handling.

## Phase 3: Component Specification & Dispatch

This is the core loop. For each section in your page topology (top to bottom), you do THREE things: **extract**, **write the spec file**, then **dispatch builders**.

### Step 1: Extract

For each section, use browser MCP to extract everything:

1. **Screenshot** the section in isolation (scroll to it, screenshot the viewport). Save to `docs/design-references/`.

2. **Extract CSS** for every element in the section. Use the extraction script below — don't hand-measure individual properties. Run it once per component container and capture the full output:

```javascript
// Per-component extraction — run via browser MCP
// Replace SELECTOR with the actual CSS selector for the component
(function(selector) {
  const el = document.querySelector(selector);
  if (!el) return JSON.stringify({ error: 'Element not found: ' + selector });
  const props = [
    'fontSize','fontWeight','fontFamily','lineHeight','letterSpacing','color',
    'textTransform','textDecoration','backgroundColor','background',
    'padding','paddingTop','paddingRight','paddingBottom','paddingLeft',
    'margin','marginTop','marginRight','marginBottom','marginLeft',
    'width','height','maxWidth','minWidth','maxHeight','minHeight',
    'display','flexDirection','justifyContent','alignItems','gap',
    'gridTemplateColumns','gridTemplateRows',
    'borderRadius','border','borderTop','borderBottom','borderLeft','borderRight',
    'boxShadow','overflow','overflowX','overflowY',
    'position','top','right','bottom','left','zIndex',
    'opacity','transform','transition','cursor',
    'objectFit','objectPosition','mixBlendMode','filter','backdropFilter',
    'whiteSpace','textOverflow','WebkitLineClamp'
  ];
  function extractStyles(element) {
    const cs = getComputedStyle(element);
    const styles = {};
    props.forEach(p => { const v = cs[p]; if (v && v !== 'none' && v !== 'normal' && v !== 'auto' && v !== '0px' && v !== 'rgba(0, 0, 0, 0)') styles[p] = v; });
    return styles;
  }
  function walk(element, depth) {
    if (depth > 4) return null;
    const children = [...element.children];
    return {
      tag: element.tagName.toLowerCase(),
      classes: element.className?.toString().split(' ').slice(0, 5).join(' '),
      text: element.childNodes.length === 1 && element.childNodes[0].nodeType === 3 ? element.textContent.trim().slice(0, 200) : null,
      styles: extractStyles(element),
      images: element.tagName === 'IMG' ? { src: element.src, alt: element.alt, naturalWidth: element.naturalWidth, naturalHeight: element.naturalHeight } : null,
      childCount: children.length,
      children: children.slice(0, 20).map(c => walk(c, depth + 1)).filter(Boolean)
    };
  }
  return JSON.stringify(walk(el, 0), null, 2);
})('SELECTOR');
```

3. **Extract multi-state styles** — for any element with multiple states (scroll-triggered, hover, active tab), capture BOTH states:

```javascript
// State A: capture styles at current state (e.g., scroll position 0)
// Then trigger the state change (scroll, click, hover via browser MCP)
// State B: re-run the extraction script on the same element
// The diff between A and B IS the behavior specification
```

Record the diff explicitly: "Property X changes from VALUE_A to VALUE_B, triggered by TRIGGER, with transition: TRANSITION_CSS."

4. **Extract real content** — all text, alt attributes, aria labels, placeholder text. Use `element.textContent` for each text node. For tabbed/stateful content, **click each tab and extract content per state**.

5. **Identify assets** this section uses — which downloaded images/videos from `public/`, which icon components from `icons.tsx`. Check for **layered images** (multiple `<img>` or background-images stacked in the same container).

6. **Assess complexity** — how many distinct sub-components does this section contain? A distinct sub-component is an element with its own unique styling, structure, and behavior (e.g., a card, a nav item, a search panel).

### Step 2: Write the Component Spec File

For each section (or sub-component, if you're breaking it up), create a spec file in `docs/research/components/`. This is NOT optional — every builder must have a corresponding spec file.

**File path:** `docs/research/components/<component-name>.spec.md`

**Template:**

```markdown
# <ComponentName> Specification

## Overview
- **Target file:** `src/components/<ComponentName>.tsx`
- **Screenshot:** `docs/design-references/<screenshot-name>.png`
- **Interaction model:** <static | click-driven | scroll-driven | time-driven>

## DOM Structure
<Describe the element hierarchy — what contains what>

## Computed Styles (exact values from getComputedStyle)

### Container
- display: ...
- padding: ...
- maxWidth: ...
- (every relevant property with exact values)

### <Child element 1>
- fontSize: ...
- color: ...
- (every relevant property)

### <Child element N>
...

## States & Behaviors

### <Behavior name, e.g., "Scroll-triggered floating mode">
- **Trigger:** <exact mechanism — scroll position 50px, IntersectionObserver rootMargin "-30% 0px", click on .tab-button, hover>
- **State A (before):** maxWidth: 100vw, boxShadow: none, borderRadius: 0
- **State B (after):** maxWidth: 1200px, boxShadow: 0 4px 20px rgba(0,0,0,0.1), borderRadius: 16px
- **Transition:** transition: all 0.3s ease
- **Implementation approach:** <CSS transition + scroll listener | IntersectionObserver | CSS animation-timeline | etc.>

### Hover states
- **<Element>:** <property>: <before> → <after>, transition: <value>

## Per-State Content (if applicable)

### State: "Featured"
- Title: "..."
- Subtitle: "..."
- Cards: [{ title, description, image, link }, ...]

### State: "Productivity"
- Title: "..."
- Cards: [...]

## Modal Variants (if this surface opens any modal/dialog)
For EACH modal: name, trigger, full DOM/layout, every form field (label + placeholder + type), validation rules + the inline error AND toast on invalid submit, and the success behaviour. Confirm/delete dialogs: the exact "Are you sure…" copy and whether custom or native `window.confirm`.

## Toast Catalog (verbatim copy — required if this surface fires any toast/alert)
For EACH action: the success toast and the error toast — copy **verbatim**, plus color, icon, position, duration. Do NOT paraphrase; these are product strings.

## Async States (loading / empty / error — required for any data-driven surface)
- **Loading:** spinner/skeleton frame.
- **Empty:** empty-state copy (verbatim) + illustration.
- **Error:** error toast copy (verbatim) + fallback UI.

## Theme Variant (if the app has a light/dark toggle)
The full second color set for this surface — every token/value that changes between light and dark.

## Assets
- Background image: `public/images/<file>.webp`
- Overlay image: `public/images/<file>.png`
- Icons used: <ArrowIcon>, <SearchIcon> from icons.tsx

## Text Content (verbatim)
<All text content, copy-pasted from the live site>

## Responsive Behavior
- **Desktop (1440px):** <layout description>
- **Tablet (768px):** <what changes — e.g., "maintains 2-column, gap reduces to 16px">
- **Mobile (390px):** <what changes — e.g., "stacks to single column, images full-width">
- **Breakpoint:** layout switches at ~<N>px
```

Fill every section. If a section doesn't apply (e.g., no states for a static footer), write "N/A" — but think twice before marking States & Behaviors as N/A. Even a footer might have hover states on links.

### Step 3: Dispatch Builders

Based on complexity, dispatch builder agent(s) in worktree(s):

**Simple section** (1-2 sub-components): One builder agent gets the entire section.

**Complex section** (3+ distinct sub-components): Break it up. One agent per sub-component, plus one agent for the section wrapper that imports them. Sub-component builders go first since the wrapper depends on them.

**What every builder agent receives:**
- The full contents of its component spec file (inline in the prompt — don't say "go read the spec file")
- Path to the section screenshot in `docs/design-references/`
- Which shared components to import (`icons.tsx`, `cn()`, shadcn primitives)
- The target file path (e.g., `src/components/HeroSection.tsx`)
- Instruction to verify with `npx tsc --noEmit` before finishing
- For responsive behavior: the specific breakpoint values and what changes
- **The Behaviour Implementation mandate (Phase 3.5):** if the spec has ANY `States & Behaviors`, `Modal Variants`, `Toast Catalog`, `Async States`, tabs, confirm dialogs, or header chrome, the builder MUST implement them as WORKING React (`"use client"`, state, wired `onClick`, conditional modal render, `toast.*` with verbatim copy, mock data) — NOT static markup. Spell this out per surface in the prompt: e.g. "the `Send SMS` button must `onClick`→`setSendOpen(true)`; render `<SendSmsModal open={sendOpen} onClose={…}>` with the captured fields; submit-empty fires `toast.error('Phone number is required')`." A builder that ships a dead button has not completed its task.

**Don't wait.** As soon as you've dispatched the builder(s) for one section, move to extracting the next section. Builders work in parallel in their worktrees while you continue extraction.

### Step 4: Merge

As builder agents complete their work:
- Merge their worktree branches into main
- You have full context on what each agent built, so resolve any conflicts intelligently
- After each merge, verify the build still passes: `npm run build`
- If a merge introduces type errors, fix them immediately

The extract → spec → dispatch → merge cycle continues until all sections are built.

## Phase 3.5: Behavior Implementation (make every captured surface actually work)

Capture (Phase 1) and static build (Phase 3) are upstream of this. This phase is where the clone stops being a screenshot and becomes a working app. For EACH interactive surface you captured, the responsible builder implements real behaviour. This is non-negotiable — it is the difference between a clone the user can click through and a dead mockup.

**The implementation contract, per surface class:**

- **Modal / dialog** — the host page is `"use client"` with `const [<name>Open, set<Name>Open] = useState(false)`. The captured trigger's `onClick` calls `set<Name>Open(true)`. Render the modal component conditionally (`{<name>Open && <Modal …/>}` or `open=` prop) with the captured title, body, and EVERY captured form field (label/placeholder/type). Backdrop click + Esc + a close button call `set<Name>Open(false)`. Internal tabs (AllClicks/AddContacts etc.) get their own `useState` active-tab.
- **Toast** — install `react-hot-toast` (pin a version compatible with the scaffold's React, e.g. `react-hot-toast@^2`; match what the original used). Mount `<Toaster position=… />` once in the layout. Every action's success AND error path calls `toast.success('<verbatim>')` / `toast.error('<verbatim>')` with the EXACT captured copy, color, icon, duration. For async submit paths that showed a pending toast, use `toast.loading(...)` (or `toast.promise`) and resolve it to success/error.
- **Button states** — a button that submits MUST show its captured pending state: swap the label (`Create`→`Creating...`) and set `disabled` while submitting (`disabled={loading}` / `disabled={loading || invalid}`), exactly as the original does it everywhere. Capture and apply all 4 visual states (default/hover/active/disabled) per the spec — a button that fires but never shows pending/disabled feedback is half-implemented.
- **Form** — controlled inputs (`useState` per field or one form-state object). On submit: run the captured client-side validation in order, firing the captured error toast for the first failure ("Phone number is required", "Invalid JSON in variables field", …); on valid submit show the captured success toast and close/redirect. No real network — validate + mutate mock state + toast.
- **Tabs / pills** — `useState` active key; clicking a tab switches the rendered panel; apply the captured active/inactive styles + transition.
- **Confirm / delete / restore** — implement the captured pattern: a custom danger modal ("Are you sure you want to delete **X**?") OR `window.confirm(...)`; confirming removes the row from the mock array and fires the captured toast.
- **Lists** — seed a realistic in-file **mock array** (5-12 rows) so the list renders populated by default. A filter/search that matches nothing → render the captured **empty state**. A `loading` boolean (optionally a fake 600ms delay on first mount) → render the captured spinner/skeleton. Pagination (`limit`/`offset`) slices the mock array.
- **Header chrome** — user dropdown (Profile/Settings/Logout) opens on click; notifications bell opens its panel (with the captured empty state); the **theme toggle** flips the real theme class on `<html>`/`documentElement` exactly as the original does (many apps add the new class AND remove the old — e.g. add `dark` + remove `light`, or vice-versa; match the captured behaviour, don't assume dark-only) so the whole page switches to the captured second color set. `next-themes` is fine (export-safe).

**Mock-data module:** create `src/lib/mock/<feature>.ts` exporting the seed arrays so list pages, modals, and stats all read from one place. Keeps pages client-interactive without any backend.

**`output:export` safety (do NOT break the build):** client components, `useState`, `react-hot-toast`, and `dark`-class theming are ALL static-export compatible. Forbidden (they break `next export` or leak the original): server actions, `fetch()` to the original backend, server-only data loading, route handlers, `next/headers`, dynamic server params. Keep every interactive page client-side + mock-data-driven.

**Per-surface done check (builder self-gate):** the builder does not finish a surface until — in its own head/`tsc` — the trigger is wired to state, the surface renders conditionally, and the captured copy is verbatim. A rendered-but-unwired trigger is an incomplete task, same as a type error. **Note: `tsc` passing does NOT mean a surface works** (a dead button type-checks fine). Builder self-report is **provisional**; the only authority that marks a surface truly done is the **Phase 5.5 functional gate**, which clicks the clone's own button and observes the result.

## Phase 4: Page Assembly

After all sections are built and merged, wire everything together in `src/app/page.tsx`:

- Import all section components
- Implement the page-level layout from your topology doc (scroll containers, column structures, sticky positioning, z-index layering)
- Connect real content to component props
- Implement page-level behaviors: scroll snap, scroll-driven animations, dark-to-light transitions, intersection observers, smooth scroll (Lenis etc.)
- Verify: `npm run build` passes clean

## Phase 5: Visual QA Diff

After assembly, do NOT declare the clone complete. Take side-by-side comparison screenshots:

1. Open the original site and your clone side-by-side (or take screenshots at the same viewport widths)
2. Compare section by section, top to bottom, at desktop (1440px)
3. Compare again at mobile (390px)
4. For each discrepancy found:
   - Check the component spec file — was the value extracted correctly?
   - If the spec was wrong: re-extract from browser MCP, update the spec, fix the component
   - If the spec was right but the builder got it wrong: fix the component to match the spec
5. Test all interactive behaviors: scroll through the page, click every button/tab, hover over interactive elements
6. Verify smooth scroll feels right, header transitions work, tab switching works, animations play

Only after this visual QA pass is the clone complete.

## Phase 5.5: Functional Gate (re-drive the CLONE's own buttons — the real completion bar)

Visual QA (Phase 5) proves it LOOKS right. This phase proves it WORKS. Run the production build/preview of YOUR CLONE and drive it with browser MCP exactly as a user would — clicking the clone's own buttons, not the original's. For each captured trigger, click it and ASSERT the captured surface actually appears:

- Click each modal trigger → the modal OPENS (assert its title/fields are in the DOM). Close it.
- Submit a form empty/invalid → the captured validation **toast fires** (assert the verbatim copy appears).
- Submit a form valid → the captured success toast fires + modal closes / redirect happens.
- Click each tab → the panel **switches** (assert different content is shown).
- Trigger a delete/confirm → the confirm appears; confirming removes the mock row.
- Open the header user dropdown + notifications panel → they open.
- Toggle the theme → the page **actually switches** light↔dark (assert a color token changed).
- Exercise pagination → the rows change.

**Coverage line (the completion metric):** per route, count
```
triggers_captured:   <N>   # interactive surfaces captured in Phase 1
triggers_functional: <M>   # surfaces that DEMONSTRABLY work when clicked on the clone (this phase)
functional_coverage: triggers_functional / triggers_captured
```
A route is complete only when `functional_coverage == 1.0`. A captured-but-dead trigger fails the gate — go back to Phase 3.5 and wire it. **Do NOT report the clone done while any captured button does nothing when clicked.** This gate is the direct answer to "the buttons don't work": they must work here, verified by clicking them, before completion.

## Pre-Dispatch Checklist

Before dispatching ANY builder agent, verify you can check every box. If you can't, go back and extract more.

- [ ] Spec file written to `docs/research/components/<name>.spec.md` with ALL sections filled
- [ ] Every CSS value in the spec is from `getComputedStyle()`, not estimated
- [ ] Interaction model is identified and documented (static / click / scroll / time)
- [ ] For stateful components: every state's content and styles are captured
- [ ] For scroll-driven components: trigger threshold, before/after styles, and transition are recorded
- [ ] For hover states: before/after values and transition timing are recorded
- [ ] All images in the section are identified (including overlays and layered compositions)
- [ ] Responsive behavior is documented for at least desktop and mobile
- [ ] Text content is verbatim from the site, not paraphrased
- [ ] The builder prompt is under ~150 lines of spec; if over, the section needs to be split
- [ ] **Behaviour mandate is in the prompt:** every captured modal/toast/tab/confirm/form/chrome surface is spelled out as a WORKING-React instruction (`"use client"` + state + wired `onClick` + conditional render + verbatim `toast.*` + mock data), not "render this button". A dead button is an incomplete build.

## What NOT to Do

These are lessons from previous failed clones — each one cost hours of rework:

- **Don't stop at the login page.** If the target is gated, authenticate via the `operator-bridge` and clone the surface BEHIND the wall — that surface is the product. A clone of only the login screen is a near-total miss.
- **Don't clone only the entry route.** Enumerate and clone EVERY reachable route (up to the 25-route cap). A single-page clone of a multi-route app reproduces a fraction of it.
- **Don't skip a modal/toast/dialog because it needs a click.** Open every modal, fire every toast (success AND error), trigger every confirm dialog, and record the copy verbatim. These are invisible to a screenshot and are most of the product's real surface.
- **Don't skip the async states.** Loading, empty, and error states are where a data app spends most of its life — force-trigger and capture each.
- **Don't build click-based tabs when the original is scroll-driven (or vice versa).** Determine the interaction model FIRST by scrolling before clicking. This is the #1 most expensive mistake — it requires a complete rewrite, not a CSS fix.
- **Don't extract only the default state.** If there are tabs showing "Featured" on load, click Productivity, Creative, Lifestyle and extract each one's cards/content. If the header changes on scroll, capture styles at position 0 AND position 100+.
- **Don't miss overlay/layered images.** A background watercolor + foreground UI mockup = 2 images. Check every container's DOM tree for multiple `<img>` elements and positioned overlays.
- **Don't build mockup components for content that's actually videos/animations.** Check if a section uses `<video>`, Lottie, or canvas before building elaborate HTML mockups of what the video shows.
- **Don't approximate CSS classes.** "It looks like `text-lg`" is wrong if the computed value is `18px` and `text-lg` is `18px/28px` but the actual line-height is `24px`. Extract exact values.
- **Don't build everything in one monolithic commit.** The whole point of this pipeline is incremental progress with verified builds at each step.
- **Don't reference docs from builder prompts.** Each builder gets the CSS spec inline in its prompt — never "see DESIGN_TOKENS.md for colors." The builder should have zero need to read external docs.
- **Don't skip asset extraction.** Without real images, videos, and fonts, the clone will always look fake regardless of how perfect the CSS is.
- **Don't give a builder agent too much scope.** If you're writing a builder prompt and it's getting long because the section is complex, that's a signal to break it into smaller tasks.
- **Don't bundle unrelated sections into one agent.** A CTA section and a footer are different components with different designs — don't hand them both to one agent and hope for the best.
- **Don't skip responsive extraction.** If you only inspect at desktop width, the clone will break at tablet and mobile. Test at 1440, 768, and 390 during extraction.
- **Don't forget smooth scroll libraries.** Check for Lenis (`.lenis` class), Locomotive Scroll, or similar. Default browser scrolling feels noticeably different and the user will spot it immediately.
- **Don't dispatch builders without a spec file.** The spec file forces exhaustive extraction and creates an auditable artifact. Skipping it means the builder gets whatever you can fit in a prompt from memory.
- **Don't ship dead buttons (the #1 functional defect).** A cloned `<button>` with no `onClick`, a page with no `"use client"`/state, a captured modal that never renders, a tab bar that doesn't switch, a form with no validation toasts, a theme toggle that does nothing — these are FAILURES even if pixel-perfect. Capturing a surface into a spec is the input, not the deliverable; the deliverable is the clone's own button WORKING when clicked (Phase 3.5 implements it, Phase 5.5 proves it).
- **Don't `fetch()` the original backend or use server-only features.** It won't exist for the clone, it breaks `output:export`, and it leaks the original's API. Drive every interactive surface from client-side mock data + `useState`.

## Completion

When done, report:
- **Authentication:** whether the app required login and whether you authenticated (or degraded to public surface because `NO_OPERATOR_RESPONSE`).
- **Route coverage:** the full ROUTE LIST from `ROUTE_MAP.md`, and confirmation that EVERY route was authored (X of N routes built). Any route skipped (and why — e.g. dynamic `:id` collapsed to one instance, or SuperAdmin out of scope).
- Total sections built
- Total components created
- Total spec files written (should match components)
- **Interaction coverage per route:** for each route, confirm the per-page checklist passed — every modal opened, every toast captured (verbatim), every confirm dialog, every tab set, header dropdown + notifications + theme toggle, and the loading/empty/error states.
- **FUNCTIONAL coverage per route (the headline bar — Phase 5.5):** `triggers_functional / triggers_captured` for each route, and the proof that captured buttons WORK when clicked on the clone (modals open, toasts fire, tabs switch, theme flips). A route is only "done" at `functional_coverage == 1.0`. Call out any trigger that still doesn't work.
- Total assets downloaded (images, videos, SVGs, fonts)
- Build status (`npm run build` result)
- Visual QA results (any remaining discrepancies)
- Any known gaps or limitations
