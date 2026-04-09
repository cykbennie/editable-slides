# Editable HTML Presentation Template

Reference architecture for generating editable slide presentations. Every presentation follows this structure.

## Base HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Presentation Title</title>

    <!-- Fonts -->
    <link rel="stylesheet" href="https://api.fontshare.com/v2/css?f[]=..." />

    <style>
        /* === CSS CUSTOM PROPERTIES (THEME) === */
        :root {
            --bg-primary: #0a0f1c;
            --bg-secondary: #111827;
            --text-primary: #ffffff;
            --text-secondary: #9ca3af;
            --accent: #00ffcc;
            --accent-glow: rgba(0, 255, 204, 0.3);
            --font-display: "Clash Display", sans-serif;
            --font-body: "Satoshi", sans-serif;
            --title-size: clamp(2rem, 6vw, 5rem);
            --subtitle-size: clamp(0.875rem, 2vw, 1.25rem);
            --body-size: clamp(0.75rem, 1.2vw, 1rem);
            --slide-padding: clamp(1.5rem, 4vw, 4rem);
            --content-gap: clamp(1rem, 2vw, 2rem);
            --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
            --duration-normal: 0.6s;
        }

        * { margin: 0; padding: 0; box-sizing: border-box; }

        /* --- PASTE viewport-base.css CONTENTS HERE --- */

        /* === ANIMATIONS === */
        .reveal {
            opacity: 0;
            transform: translateY(30px);
            transition: opacity var(--duration-normal) var(--ease-out-expo),
                        transform var(--duration-normal) var(--ease-out-expo);
        }
        .slide.visible .reveal {
            opacity: 1;
            transform: translateY(0);
        }

        /* === EDITOR UI STYLES === */
        /* These are hidden by default, shown only in edit mode */
        .edit-toolbar,
        .slide-panel,
        .edit-handle,
        .resize-handle,
        .element-outline {
            display: none !important;
        }

        body.edit-active .edit-toolbar,
        body.edit-active .slide-panel {
            display: flex !important;
        }

        body.edit-active .slide-content > *:hover > .edit-handle,
        body.edit-active .slide-content > *.selected > .edit-handle {
            display: block !important;
        }

        body.edit-active .slide img:hover > .resize-handle,
        body.edit-active .slide img.selected > .resize-handle {
            display: block !important;
        }

        /* ... preset-specific styles ... */
    </style>
</head>
<body>
    <!-- Progress bar (presentation mode only) -->
    <div class="progress-bar"></div>

    <!-- Navigation dots (presentation mode only) -->
    <nav class="nav-dots"></nav>

    <!-- Edit mode UI (hidden by default) -->
    <div class="edit-toolbar" id="editToolbar">
        <!-- Toolbar populated by JS -->
    </div>

    <!-- Slide management panel (edit mode only) -->
    <div class="slide-panel" id="slidePanel">
        <!-- Thumbnails populated by JS -->
    </div>

    <!-- Slides -->
    <section class="slide title-slide" data-slide-index="0">
        <div class="slide-content">
            <h1 class="reveal editable-element">Presentation Title</h1>
            <p class="reveal editable-element">Subtitle or author</p>
        </div>
    </section>

    <section class="slide" data-slide-index="1">
        <div class="slide-content">
            <h2 class="reveal editable-element">Slide Title</h2>
            <p class="reveal editable-element">Content...</p>
        </div>
    </section>

    <script>
        /* === SLIDE PRESENTATION CONTROLLER === */
        class SlidePresentation { /* ... same as frontend-slides ... */ }

        /* === EDITOR CONTROLLER === */
        class SlideEditor { /* ... see editor-system.md ... */ }

        const presentation = new SlidePresentation();
        const editor = new SlideEditor(presentation);
    </script>
</body>
</html>
```

## Key Differences from Frontend-Slides Template

1. **`data-slide-index`** on each `<section class="slide">` — enables slide reordering
2. **`.editable-element`** class on all text/image elements — marks them as draggable and editable
3. **Edit UI containers** in the DOM but hidden via CSS by default
4. **`SlideEditor` class** alongside `SlidePresentation` — manages all editing logic
5. **CSS rule**: `body.edit-active` toggles visibility of all editing UI

## Element Marking

Every text element and image that users should be able to edit must have:

- `class="editable-element"` — marks it for the editor system
- Text elements additionally get `contenteditable` when entering edit mode
- Images get resize handles when in edit mode

The editor system automatically adds these behaviors when the user enters edit mode — you do NOT need to add `contenteditable` attributes in the generated HTML. Just add the `editable-element` class.

## Image Handling

Same as frontend-slides: `max-height: min(50vh, 400px)`, direct file paths (not base64), processed with Pillow if needed. In edit mode, images get resize handles that allow users to change dimensions.

## Code Quality

- Semantic HTML (`<section>`, `<nav>`, `<main>`)
- Full keyboard navigation in both modes
- ARIA labels on editing UI elements
- `prefers-reduced-motion` support (in viewport-base.css)
- Clear section comments throughout

## File Structure

```
presentation.html    # Self-contained, all CSS/JS inline
assets/              # Images only, if any
```
