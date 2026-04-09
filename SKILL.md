---
name: editable-slides
description: Create stunning, animation-rich HTML presentations that are fully editable directly in the browser — like a mini Google Slides. Use whenever the user wants to build a presentation, create slides, make a pitch deck, or convert a PPT/PPTX to web AND wants to be able to edit text, images, and slide structure in the browser after generation. Also trigger when users ask for "editable slides", "interactive slides", "live-editable presentation", or "slides I can modify in browser". This is a superset of frontend-slides — if the user wants any kind of browser-based editing on their slides, use this skill.
---

# Editable Slides

Create zero-dependency, animation-rich HTML presentations with a **built-in visual editor** that runs entirely in the browser. Users can edit text, move/resize elements, add/delete images, add/delete/reorder slides — all without touching code.

## Core Principles

1. **Zero Dependencies** — Single HTML file with inline CSS/JS. No npm, no build tools.
2. **Show, Don't Tell** — Generate visual previews for style discovery, not abstract choices.
3. **Distinctive Design** — No generic "AI slop." Every presentation must feel custom-crafted.
4. **Viewport Fitting (NON-NEGOTIABLE)** — Every slide MUST fit exactly within 100vh in presentation mode.
5. **Full Editability** — Every text and image element is draggable, resizable, and editable in-browser.
6. **Presentation ↔ Edit Mode** — Seamless toggle between presenting and editing. Edit mode shows drag handles, toolbar, and slide panel; presentation mode shows only the polished slides.

## Architecture

The generated HTML has two modes controlled by a single toggle:

- **Presentation Mode** (default) — Full-screen slides with scroll-snap, animations, navigation. Identical to frontend-slides output.
- **Edit Mode** — A floating toolbar appears at the top, a slide thumbnail panel on the left, elements get drag handles and resize handles, and clicking text makes it editable.

The key insight: edit mode is a **layer on top** of the presentation. The same DOM serves both modes. CSS classes (`body.edit-active`) control visibility of editing UI. JavaScript adds/removes event listeners when toggling modes.

## Design Aesthetics

Follow the same design philosophy as frontend-slides — distinctive, custom-crafted, anti-AI-slop. Read [STYLE_PRESETS.md](references/STYLE_PRESETS.md) for the 12 curated presets.

Focus on:
- Typography: Beautiful, unique fonts (Fontshare or Google Fonts). Avoid Inter, Roboto, Arial.
- Color: Cohesive palette with CSS custom properties. Dominant colors with sharp accents.
- Motion: CSS-first animations with staggered reveals for delight.
- Backgrounds: Layered gradients, geometric patterns, contextual effects — never flat solid colors.

## Viewport Fitting Rules

These apply to **presentation mode**. In edit mode, scroll-snap is disabled and slides can be scrolled through.

- Every `.slide` must have `height: 100vh; height: 100dvh; overflow: hidden;`
- ALL font sizes use `clamp(min, preferred, max)`
- Images: `max-height: min(50vh, 400px)`
- Breakpoints: 700px, 600px, 500px heights
- Include `prefers-reduced-motion` support

**When generating, read [references/viewport-base.css](references/viewport-base.css) and include its full contents.**

### Content Density Limits (Presentation Mode)

| Slide Type    | Max Content                                                |
| ------------- | ---------------------------------------------------------- |
| Title slide   | 1 heading + 1 subtitle + optional tagline                  |
| Content slide | 1 heading + 4-6 bullets OR 1 heading + 2 paragraphs        |
| Feature grid  | 1 heading + 6 cards max (2x3 or 3x2)                      |
| Code slide    | 1 heading + 8-10 lines of code                             |
| Quote slide   | 1 quote (max 3 lines) + attribution                        |
| Image slide   | 1 heading + 1 image (max 60vh height)                      |

In edit mode users can add more content — that's their choice. The limits are for initial generation to ensure the presentation looks good.

---

## Phase 0: Detect Mode

- **Mode A: New Presentation** — Create from scratch. Go to Phase 1.
- **Mode B: PPT Conversion** — Convert a .pptx file. Go to Phase 4.
- **Mode C: Enhancement** — Improve an existing editable-slides presentation. Read it, understand it, enhance. **Follow Mode C modification rules below.**

### Mode C: Modification Rules

When enhancing existing editable presentations, preserve the editing system and watch for viewport fitting:

1. **Preserve the editor** — Never remove the SlideEditor class, toolbar, slide panel, or edit toggle. Any modification must keep the full editing system intact.
2. **Mark new content** — Any new text or images you add must have `class="editable-element"` so they're editable.
3. **Before adding content:** Count existing elements on the target slide, check against density limits
4. **Adding images:** Must have `max-height: min(50vh, 400px)`. If slide already has max content, split into two slides
5. **Adding text:** Max 4-6 bullets per slide. Exceeds limits? Split into continuation slides
6. **After ANY modification, verify:** `.slide` has `overflow: hidden`, new elements use `clamp()`, new elements have `editable-element` class, images have viewport-relative max-height
7. **Proactively reorganize:** If modifications will cause overflow, automatically split content and inform the user

---

## Phase 1: Content Discovery

Ask ALL questions in a single AskUserQuestion call:

**Question 1 — Purpose** (header: "Purpose"):
What is this presentation for? Options: Pitch deck / Teaching-Tutorial / Conference talk / Internal presentation

**Question 2 — Length** (header: "Length"):
Approximately how many slides? Options: Short 5-10 / Medium 10-20 / Long 20+

**Question 3 — Content** (header: "Content"):
Do you have content ready? Options: All content ready / Rough notes / Topic only

If user has content, ask them to share it.

### Step 1.2: Image Evaluation (if images provided)

Same as frontend-slides: scan, view, evaluate, co-design outline with curated images.

---

## Phase 2: Style Discovery

Same as frontend-slides. Generate 3 visual previews based on mood. Read [references/STYLE_PRESETS.md](references/STYLE_PRESETS.md) for presets.

### Step 2.1: Mood Selection

Ask (header: "Vibe", multiSelect: true, max 2): What feeling should the audience have?
Options: Impressed/Confident / Excited/Energized / Calm/Focused / Inspired/Moved

### Step 2.2: Generate 3 Style Previews

Save to `.claude-design/slide-previews/` (style-a.html, style-b.html, style-c.html). Open each for the user.

### Step 2.3: User Picks

Ask which style they prefer.

---

## Phase 3: Generate Presentation

Generate the full presentation with the **built-in editing system**.

**Before generating, read these files:**

- [references/html-template.md](references/html-template.md) — Editable HTML architecture, JS features
- [references/viewport-base.css](references/viewport-base.css) — Mandatory responsive CSS
- [references/editor-system.md](references/editor-system.md) — **CRITICAL: The complete inline editor implementation guide** — toolbar, drag system, slide management, save/export. This is the core of what makes this skill different from frontend-slides.
- [references/animation-patterns.md](references/animation-patterns.md) — Animation reference

**Key requirements:**

- Single self-contained HTML file, all CSS/JS inline
- Include the FULL contents of viewport-base.css
- Include the FULL editing system from editor-system.md
- Use fonts from Fontshare or Google Fonts — never system fonts
- Add detailed comments explaining each section

---

## Phase 4: PPT Conversion

When converting PowerPoint files:

1. **Extract content** — Run `python scripts/extract-pptx.py <input.pptx> <output_dir>`
2. **Confirm with user** — Present extracted slide titles, content summaries, image counts
3. **Style selection** — Proceed to Phase 2
4. **Generate HTML** — Convert to chosen style WITH the full editing system

---

## Phase 5: Delivery

1. **Clean up** — Delete `.claude-design/slide-previews/` if it exists
2. **Open** — Use `open [filename].html` to launch in browser
3. **Summarize** — Tell the user:
   - File location, style name, slide count
   - **Presentation mode:** Arrow keys, Space, scroll/swipe, nav dots
   - **Edit mode:** Press `E` or click the pencil icon (top-left) to enter edit mode
   - **In edit mode:**
     - Click any text to edit it
     - Drag any element to reposition it
     - Resize images via corner handles
     - Use the toolbar for formatting (bold, italic, underline, font size, color, alignment, lists)
     - Add text boxes or images via toolbar buttons
     - Add/delete/duplicate slides via the slide panel on the left
     - Drag slides in the panel to reorder
     - Press `Ctrl+S` to save (downloads clean HTML — no edit UI in the saved file)
   - How to customize CSS variables for theme changes

---

## Phase 6: Share & Export (Optional)

After delivery, ask: "Would you like to share this presentation? I can deploy it to a live URL or export it as a PDF."

### 6A: Deploy to Vercel

Run `bash scripts/deploy.sh <path-to-presentation>`. Guide the user through Vercel setup if needed.

### 6B: Export to PDF

Run `bash scripts/export-pdf.sh <path-to-html> [output.pdf]`.

---

## Supporting Files

| File | Purpose | When to Read |
|------|---------|--------------|
| [references/STYLE_PRESETS.md](references/STYLE_PRESETS.md) | 12 curated visual presets | Phase 2 |
| [references/viewport-base.css](references/viewport-base.css) | Mandatory responsive CSS | Phase 3 |
| [references/html-template.md](references/html-template.md) | Editable HTML architecture | Phase 3 |
| [references/editor-system.md](references/editor-system.md) | **The editor implementation guide** | Phase 3 |
| [references/animation-patterns.md](references/animation-patterns.md) | CSS/JS animation reference | Phase 3 |
| [scripts/extract-pptx.py](scripts/extract-pptx.py) | PPT content extraction | Phase 4 |
| [scripts/deploy.sh](scripts/deploy.sh) | Deploy to Vercel | Phase 6 |
| [scripts/export-pdf.sh](scripts/export-pdf.sh) | Export to PDF | Phase 6 |
