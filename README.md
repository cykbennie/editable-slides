# editable-slides

A Claude Code skill that creates **stunning, animation-rich HTML presentations fully editable directly in the browser** — like a mini Google Slides.

## Features

- **Single HTML file** — Zero dependencies, all CSS/JS inline
- **Rich visual editor** — Toolbar with bold/italic/underline/font size/color/alignment/lists
- **Drag & drop** — Freely reposition any text or image element on a slide
- **Image resize** — Corner handles with aspect ratio lock (shift to unlock)
- **Slide management** — Add, delete, duplicate, and reorder slides via sidebar panel
- **Image insert** — Add images via URL or file upload
- **Text insert** — Add new text boxes to any slide
- **Auto-save** — Persists edits to localStorage every 5 seconds
- **Clean export** — Ctrl+S downloads HTML with all edit UI stripped
- **12 style presets** — Curated visual themes (Bold Signal, Dark Botanical, Notebook Tabs, etc.)
- **Viewport-perfect** — Every slide fits exactly in 100vh with responsive breakpoints
- **PPT conversion** — Convert .pptx files to editable HTML presentations
- **Share & export** — Deploy to Vercel or export to PDF

## Installation

Add to your Claude Code skills directory:

```bash
cp -r SKILL.md references/ scripts/ ~/.claude/skills/editable-slides/
```

## Usage

In Claude Code, simply ask:

- "Create a 10-slide pitch deck for my startup"
- "Make a tutorial presentation about React"
- "Convert my presentation.pptx to an editable web presentation"

The skill will guide you through content discovery, style selection (with live previews), and generation.

### Edit Mode

After generation, press **E** or hover the top-left corner to enter edit mode:

- Click any text to edit it
- Drag elements to reposition
- Resize images via corner handles
- Use the toolbar for formatting
- Add/delete/reorder slides via the left panel
- **Ctrl+S** to save a clean copy

## Structure

```
editable-slides/
├── SKILL.md                  # Skill definition (trigger + workflow)
├── references/
│   ├── STYLE_PRESETS.md      # 12 curated visual styles
│   ├── viewport-base.css     # Mandatory responsive CSS
│   ├── html-template.md      # Editable HTML architecture
│   ├── editor-system.md      # Editor implementation guide
│   └── animation-patterns.md # Animation reference
└── scripts/
    ├── extract-pptx.py       # PPT content extraction
    ├── deploy.sh             # Deploy to Vercel
    └── export-pdf.sh         # Export to PDF
```

## License

MIT
