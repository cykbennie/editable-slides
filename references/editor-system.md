# Editor System Implementation Guide

This document describes the complete inline editing system that gets embedded into every generated presentation. The system enables users to edit text, drag/resize elements, manage slides, and export — all in the browser.

**This is the core differentiator from frontend-slides. Include the full implementation in every generated HTML file.**

## Architecture Overview

The editor has these components:

1. **SlideEditor class** — Main controller: toggles edit mode, manages selection, delegates to subsystems
2. **Toolbar** — Rich formatting bar (bold, italic, underline, font size, color, alignment, lists)
3. **Drag System** — Click-and-drag to reposition any element freely on its slide
4. **Resize System** — Corner handles on images to resize them
5. **Slide Panel** — Left sidebar with thumbnails for add/delete/reorder/duplicate
6. **Image Insert** — Add images via file picker or URL
7. **Text Insert** — Add new text boxes to any slide
8. **Save/Export** — Ctrl+S saves a clean HTML file (no edit UI in the saved output)

## Table of Contents

1. [Edit Mode Toggle](#edit-mode-toggle)
2. [Toolbar](#toolbar)
3. [Drag System](#drag-system)
4. [Resize System](#resize-system)
5. [Slide Panel](#slide-panel)
6. [Image Insert](#image-insert)
7. [Text Insert](#text-insert)
8. [Save/Export](#saveexport)
9. [Auto-save to localStorage](#auto-save-to-localstorage)
10. [Complete Implementation](#complete-implementation)

---

## Edit Mode Toggle

Three ways to enter/exit edit mode:

1. **Click the pencil button** — Fixed position, top-left corner
2. **Press E key** — Only when not editing text (check `contenteditable`)
3. **Click the edit hotzone** — Top-left 80x80px invisible area

When entering edit mode:
- Add `edit-active` class to `document.body`
- Disable scroll-snap (`html { scroll-snap-type: none }`)
- Make all `.editable-element` text nodes `contenteditable="true"`
- Show toolbar and slide panel
- Hide presentation-only elements (progress bar, nav dots)

When exiting edit mode:
- Remove `edit-active` class
- Re-enable scroll-snap
- Remove `contenteditable` from all elements
- Deselect everything
- Hide toolbar and slide panel
- Show presentation elements

### CSS for Toggle Button

```css
/* Edit hotzone — invisible click target, top-left */
.edit-hotzone {
    position: fixed;
    top: 0;
    left: 0;
    width: 80px;
    height: 80px;
    z-index: 10000;
    cursor: pointer;
}

/* Pencil toggle button */
.edit-toggle {
    position: fixed;
    top: 16px;
    left: 16px;
    width: 40px;
    height: 40px;
    border-radius: 50%;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.2);
    cursor: pointer;
    font-size: 18px;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.3s ease;
    z-index: 10001;
    backdrop-filter: blur(8px);
}
.edit-toggle:hover {
    background: rgba(0, 0, 0, 0.9);
    transform: scale(1.1);
}
.edit-toggle.show {
    opacity: 1;
    pointer-events: auto;
}
.edit-toggle.active {
    opacity: 1;
    pointer-events: auto;
    background: var(--accent, #00ffcc);
    color: #000;
}
```

### JS for Toggle (with 400ms grace period)

```javascript
// Hotzone hover with 400ms grace period
const hotzone = document.querySelector('.edit-hotzone');
const editToggle = document.getElementById('editToggle');
let hideTimeout = null;

hotzone.addEventListener('mouseenter', () => {
    clearTimeout(hideTimeout);
    editToggle.classList.add('show');
});
hotzone.addEventListener('mouseleave', () => {
    hideTimeout = setTimeout(() => {
        if (!editor.isActive) editToggle.classList.remove('show');
    }, 400);
});
editToggle.addEventListener('mouseenter', () => clearTimeout(hideTimeout));
editToggle.addEventListener('mouseleave', () => {
    hideTimeout = setTimeout(() => {
        if (!editor.isActive) editToggle.classList.remove('show');
    }, 400);
});
editToggle.addEventListener('click', () => editor.toggleEditMode());
hotzone.addEventListener('click', () => editor.toggleEditMode());

// Keyboard shortcut
document.addEventListener('keydown', (e) => {
    if ((e.key === 'e' || e.key === 'E') && !e.target.getAttribute('contenteditable')) {
        e.preventDefault();
        editor.toggleEditMode();
    }
});
```

---

## Toolbar

A floating toolbar at the top of the viewport in edit mode. Contains formatting controls for the currently selected text element.

### Toolbar Layout

```
[Bold] [Italic] [Underline] | [Font Size ▼] | [Text Color 🎨] | [Align L] [Align C] [Align R] | [Bullet] [Numbered] | [+ Text] [+ Image] | [Save 💾] [Present ▶]
```

### Toolbar CSS

```css
.edit-toolbar {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 48px;
    background: rgba(30, 30, 40, 0.95);
    backdrop-filter: blur(12px);
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    display: flex;
    align-items: center;
    padding: 0 12px;
    gap: 4px;
    z-index: 9999;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.toolbar-group {
    display: flex;
    align-items: center;
    gap: 2px;
    padding: 0 6px;
}

.toolbar-divider {
    width: 1px;
    height: 24px;
    background: rgba(255, 255, 255, 0.15);
    margin: 0 4px;
}

.toolbar-btn {
    width: 32px;
    height: 32px;
    border: none;
    background: transparent;
    color: rgba(255, 255, 255, 0.7);
    border-radius: 6px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 14px;
    transition: all 0.15s ease;
}
.toolbar-btn:hover {
    background: rgba(255, 255, 255, 0.1);
    color: white;
}
.toolbar-btn.active {
    background: var(--accent, #00ffcc);
    color: #000;
}

.toolbar-select {
    height: 28px;
    padding: 0 8px;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.15);
    border-radius: 6px;
    color: white;
    font-size: 12px;
    cursor: pointer;
}
.toolbar-select option {
    background: #1e1e28;
    color: white;
}

.toolbar-color-input {
    width: 28px;
    height: 28px;
    border: 2px solid rgba(255, 255, 255, 0.2);
    border-radius: 6px;
    cursor: pointer;
    padding: 0;
    background: none;
}
.toolbar-color-input::-webkit-color-swatch-wrapper { padding: 2px; }
.toolbar-color-input::-webkit-color-swatch { border-radius: 3px; border: none; }
```

### Toolbar HTML

```html
<div class="edit-toolbar" id="editToolbar">
    <div class="toolbar-group">
        <button class="toolbar-btn" data-command="bold" title="Bold (Ctrl+B)"><b>B</b></button>
        <button class="toolbar-btn" data-command="italic" title="Italic (Ctrl+I)"><i>I</i></button>
        <button class="toolbar-btn" data-command="underline" title="Underline (Ctrl+U)"><u>U</u></button>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-group">
        <select class="toolbar-select" id="fontSizeSelect" title="Font Size">
            <option value="1">12px</option>
            <option value="2">14px</option>
            <option value="3" selected>16px</option>
            <option value="4">18px</option>
            <option value="5">24px</option>
            <option value="6">32px</option>
            <option value="7">48px</option>
        </select>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-group">
        <input type="color" class="toolbar-color-input" id="textColorPicker" value="#ffffff" title="Text Color">
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-group">
        <button class="toolbar-btn" data-command="justifyLeft" title="Align Left">&#8676;</button>
        <button class="toolbar-btn" data-command="justifyCenter" title="Align Center">&#8596;</button>
        <button class="toolbar-btn" data-command="justifyRight" title="Align Right">&#8594;</button>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-group">
        <button class="toolbar-btn" data-command="insertUnorderedList" title="Bullet List">•≡</button>
        <button class="toolbar-btn" data-command="insertOrderedList" title="Numbered List">1.</button>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-group">
        <button class="toolbar-btn" id="addTextBtn" title="Add Text Box">+T</button>
        <button class="toolbar-btn" id="addImageBtn" title="Add Image">+🖼</button>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-group" style="margin-left: auto;">
        <button class="toolbar-btn" id="saveBtn" title="Save (Ctrl+S)">💾</button>
        <button class="toolbar-btn" id="presentBtn" title="Exit Edit Mode">▶</button>
    </div>
</div>
```

### Toolbar JS

```javascript
// Formatting commands (uses document.execCommand for contenteditable)
document.querySelectorAll('.toolbar-btn[data-command]').forEach(btn => {
    btn.addEventListener('click', (e) => {
        e.preventDefault();
        document.execCommand(btn.dataset.command, false, null);
        updateToolbarState();
    });
});

// Font size
document.getElementById('fontSizeSelect').addEventListener('change', (e) => {
    document.execCommand('fontSize', false, e.target.value);
});

// Text color
document.getElementById('textColorPicker').addEventListener('input', (e) => {
    document.execCommand('foreColor', false, e.target.value);
});

// Update toolbar button states to reflect current selection
function updateToolbarState() {
    document.querySelectorAll('.toolbar-btn[data-command]').forEach(btn => {
        const cmd = btn.dataset.command;
        if (['bold', 'italic', 'underline', 'justifyLeft', 'justifyCenter',
             'justifyRight', 'insertUnorderedList', 'insertOrderedList'].includes(cmd)) {
            btn.classList.toggle('active', document.queryCommandState(cmd));
        }
    });
}

// Listen for selection changes to update toolbar
document.addEventListener('selectionchange', updateToolbarState);
```

---

## Drag System

Enables free repositioning of any `.editable-element` within its parent slide.

### How It Works

When in edit mode and the user clicks an element (that is not in text-editing mode):
1. The element gets a dashed outline (`.selected`)
2. A drag handle appears at the top center
3. The user can click and drag the element to any position within the slide

Position is stored via `style="position: absolute; left: Xpx; top: Ypx;"` on the element. When first dragged, the element is taken out of normal flow and positioned absolutely within its `.slide-content` parent.

### CSS

```css
/* Selected element outline */
body.edit-active .editable-element.selected {
    outline: 2px dashed var(--accent, #00ffcc);
    outline-offset: 4px;
}

body.edit-active .editable-element:hover:not(.editing) {
    outline: 1px dashed rgba(255, 255, 255, 0.3);
    outline-offset: 4px;
    cursor: move;
}

/* Drag handle */
.drag-handle {
    display: none;
    position: absolute;
    top: -12px;
    left: 50%;
    transform: translateX(-50%);
    width: 24px;
    height: 12px;
    background: var(--accent, #00ffcc);
    border-radius: 0 0 6px 6px;
    cursor: grab;
    z-index: 100;
}

body.edit-active .editable-element.selected .drag-handle {
    display: block;
}

.drag-handle:active { cursor: grabbing; }
```

### Drag JS

```javascript
initDrag() {
    let isDragging = false;
    let currentEl = null;
    let startX, startY, startLeft, startTop;

    document.addEventListener('mousedown', (e) => {
        if (!this.isActive) return;
        const handle = e.target.closest('.drag-handle');
        if (!handle) return;
        e.preventDefault();

        currentEl = handle.parentElement;
        const slideContent = currentEl.closest('.slide-content');
        const slideRect = slideContent.getBoundingClientRect();

        // First drag: switch from static to absolute positioning
        if (!currentEl.style.position || currentEl.style.position !== 'absolute') {
            const elRect = currentEl.getBoundingClientRect();
            currentEl.style.position = 'absolute';
            currentEl.style.left = (elRect.left - slideRect.left) + 'px';
            currentEl.style.top = (elRect.top - slideRect.top) + 'px';
            currentEl.style.margin = '0';
        }

        isDragging = true;
        startX = e.clientX;
        startY = e.clientY;
        startLeft = parseInt(currentEl.style.left);
        startTop = parseInt(currentEl.style.top);
        currentEl.style.zIndex = '100';
        currentEl.style.opacity = '0.8';
    });

    document.addEventListener('mousemove', (e) => {
        if (!isDragging || !currentEl) return;
        e.preventDefault();
        const dx = e.clientX - startX;
        const dy = e.clientY - startY;
        currentEl.style.left = (startLeft + dx) + 'px';
        currentEl.style.top = (startTop + dy) + 'px';
    });

    document.addEventListener('mouseup', () => {
        if (currentEl) {
            currentEl.style.zIndex = '';
            currentEl.style.opacity = '';
        }
        isDragging = false;
        currentEl = null;
    });
}
```

---

## Resize System

Corner handles on images for resizing. Maintains aspect ratio by default, with shift-key to unlock.

### CSS

```css
/* Resize handles (only for images) */
.resize-handle {
    display: none;
    position: absolute;
    width: 12px;
    height: 12px;
    background: var(--accent, #00ffcc);
    border: 2px solid #fff;
    border-radius: 2px;
    z-index: 101;
}

.resize-handle.nw { top: -6px; left: -6px; cursor: nw-resize; }
.resize-handle.ne { top: -6px; right: -6px; cursor: ne-resize; }
.resize-handle.sw { bottom: -6px; left: -6px; cursor: sw-resize; }
.resize-handle.se { bottom: -6px; right: -6px; cursor: se-resize; }

body.edit-active .editable-element.selected img,
body.edit-active img.editable-element.selected {
    outline: 2px dashed var(--accent, #00ffcc);
    outline-offset: 2px;
}
body.edit-active img.editable-element.selected .resize-handle,
body.edit-active img.editable-element.selected ~ .resize-handle {
    display: block;
}
```

### Resize JS

```javascript
initResize() {
    let isResizing = false;
    let currentImg = null;
    let startX, startY, startW, startH;
    let aspectRatio = 1;

    document.addEventListener('mousedown', (e) => {
        if (!this.isActive) return;
        const handle = e.target.closest('.resize-handle');
        if (!handle) return;
        e.preventDefault();

        currentImg = handle.closest('img.editable-element') ||
                     handle.previousElementSibling;
        if (!currentImg || currentImg.tagName !== 'IMG') return;

        isResizing = true;
        aspectRatio = currentImg.naturalWidth / currentImg.naturalHeight;
        startX = e.clientX;
        startY = e.clientY;
        startW = currentImg.offsetWidth;
        startH = currentImg.offsetHeight;
        currentImg.classList.add('selected');
    });

    document.addEventListener('mousemove', (e) => {
        if (!isResizing || !currentImg) return;
        e.preventDefault();

        const dx = e.clientX - startX;
        const dy = e.clientY - startY;
        const maintainAspect = !e.shiftKey;

        let newW, newH;
        if (maintainAspect) {
            newW = Math.max(50, startW + dx);
            newH = newW / aspectRatio;
        } else {
            newW = Math.max(50, startW + dx);
            newH = Math.max(50, startH + dy);
        }

        currentImg.style.width = newW + 'px';
        currentImg.style.height = newH + 'px';
        currentImg.style.maxWidth = 'none';
        currentImg.style.maxHeight = 'none';
    });

    document.addEventListener('mouseup', () => {
        if (isResizing) {
            currentImg?.classList.remove('selected');
        }
        isResizing = false;
        currentImg = null;
    });
}
```

---

## Slide Panel

Left sidebar showing slide thumbnails. Users can add, delete, duplicate, and reorder slides.

### CSS

```css
.slide-panel {
    position: fixed;
    left: 0;
    top: 48px; /* Below toolbar */
    bottom: 0;
    width: 200px;
    background: rgba(20, 20, 30, 0.95);
    backdrop-filter: blur(12px);
    border-right: 1px solid rgba(255, 255, 255, 0.1);
    overflow-y: auto;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    z-index: 9998;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.slide-panel::-webkit-scrollbar {
    width: 4px;
}
.slide-panel::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
}

.slide-thumb {
    position: relative;
    aspect-ratio: 16 / 9;
    background: rgba(255, 255, 255, 0.05);
    border: 2px solid rgba(255, 255, 255, 0.1);
    border-radius: 6px;
    cursor: pointer;
    overflow: hidden;
    transition: border-color 0.2s;
}
.slide-thumb:hover {
    border-color: rgba(255, 255, 255, 0.3);
}
.slide-thumb.active {
    border-color: var(--accent, #00ffcc);
}

.slide-thumb-label {
    position: absolute;
    bottom: 4px;
    left: 6px;
    font-size: 10px;
    color: rgba(255, 255, 255, 0.6);
    pointer-events: none;
}

.slide-thumb-actions {
    position: absolute;
    top: 2px;
    right: 2px;
    display: none;
    gap: 2px;
}
.slide-thumb:hover .slide-thumb-actions {
    display: flex;
}
.slide-thumb-btn {
    width: 20px;
    height: 20px;
    border: none;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    border-radius: 4px;
    cursor: pointer;
    font-size: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
}
.slide-thumb-btn:hover {
    background: rgba(255, 50, 50, 0.8);
}
.slide-thumb-btn.duplicate:hover {
    background: rgba(50, 150, 255, 0.8);
}

.panel-add-slide {
    aspect-ratio: 16 / 9;
    border: 2px dashed rgba(255, 255, 255, 0.2);
    border-radius: 6px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: rgba(255, 255, 255, 0.4);
    font-size: 24px;
    cursor: pointer;
    transition: all 0.2s;
}
.panel-add-slide:hover {
    border-color: var(--accent, #00ffcc);
    color: var(--accent, #00ffcc);
}
```

### Slide Panel HTML

```html
<div class="slide-panel" id="slidePanel">
    <!-- Populated dynamically by JS -->
</div>
```

### Slide Panel JS

```javascript
buildSlidePanel() {
    const panel = document.getElementById('slidePanel');
    panel.innerHTML = '';

    const slides = document.querySelectorAll('.slide');
    slides.forEach((slide, i) => {
        const thumb = document.createElement('div');
        thumb.className = 'slide-thumb' + (i === this.currentSlideIndex ? ' active' : '');
        thumb.innerHTML = `
            <span class="slide-thumb-label">${i + 1}</span>
            <div class="slide-thumb-actions">
                <button class="slide-thumb-btn duplicate" data-action="duplicate" data-index="${i}" title="Duplicate">⧉</button>
                <button class="slide-thumb-btn" data-action="delete" data-index="${i}" title="Delete">✕</button>
            </div>
        `;
        thumb.addEventListener('click', (e) => {
            if (e.target.closest('.slide-thumb-btn')) return;
            this.scrollToSlide(i);
        });
        panel.appendChild(thumb);
    });

    // Add slide button
    const addBtn = document.createElement('div');
    addBtn.className = 'panel-add-slide';
    addBtn.textContent = '+';
    addBtn.addEventListener('click', () => this.addSlide());
    panel.appendChild(addBtn);

    // Delete and duplicate handlers
    panel.querySelectorAll('.slide-thumb-btn').forEach(btn => {
        btn.addEventListener('click', (e) => {
            e.stopPropagation();
            const index = parseInt(btn.dataset.index);
            if (btn.dataset.action === 'delete') this.deleteSlide(index);
            if (btn.dataset.action === 'duplicate') this.duplicateSlide(index);
        });
    });
}

scrollToSlide(index) {
    const slides = document.querySelectorAll('.slide');
    if (index >= 0 && index < slides.length) {
        slides[index].scrollIntoView({ behavior: 'smooth' });
        this.currentSlideIndex = index;
        this.buildSlidePanel();
    }
}

addSlide() {
    const slides = document.querySelectorAll('.slide');
    const newSlide = document.createElement('section');
    newSlide.className = 'slide';
    newSlide.setAttribute('data-slide-index', slides.length);
    newSlide.innerHTML = `
        <div class="slide-content">
            <h2 class="editable-element reveal visible">New Slide</h2>
            <p class="editable-element reveal visible">Click to add content</p>
        </div>
    `;
    document.body.appendChild(newSlide);
    this.buildSlidePanel();
    this.scrollToSlide(slides.length);
}

deleteSlide(index) {
    const slides = document.querySelectorAll('.slide');
    if (slides.length <= 1) return; // Don't delete last slide
    if (confirm('Delete this slide?')) {
        slides[index].remove();
        // Re-index
        document.querySelectorAll('.slide').forEach((s, i) => {
            s.setAttribute('data-slide-index', i);
        });
        this.buildSlidePanel();
    }
}

duplicateSlide(index) {
    const slides = document.querySelectorAll('.slide');
    const clone = slides[index].cloneNode(true);
    // Clear any active states
    clone.querySelectorAll('.selected').forEach(el => el.classList.remove('selected'));
    clone.querySelectorAll('[contenteditable]').forEach(el => el.removeAttribute('contenteditable'));
    document.body.insertBefore(clone, slides[index].nextSibling);
    // Re-index
    document.querySelectorAll('.slide').forEach((s, i) => {
        s.setAttribute('data-slide-index', i);
    });
    this.buildSlidePanel();
}
```

---

## Image Insert

Add images to slides via file picker or URL input.

```javascript
initImageInsert() {
    const addImageBtn = document.getElementById('addImageBtn');
    addImageBtn.addEventListener('click', () => {
        // Create a simple modal
        const modal = document.createElement('div');
        modal.style.cssText = `
            position: fixed; top: 0; left: 0; right: 0; bottom: 0;
            background: rgba(0,0,0,0.7); z-index: 10002;
            display: flex; align-items: center; justify-content: center;
        `;
        modal.innerHTML = `
            <div style="background: #1e1e28; border-radius: 12px; padding: 24px; min-width: 400px; max-width: 500px;">
                <h3 style="color: white; margin-bottom: 16px;">Add Image</h3>
                <div style="margin-bottom: 12px;">
                    <label style="color: rgba(255,255,255,0.6); font-size: 12px; display: block; margin-bottom: 4px;">From URL</label>
                    <input id="imageUrlInput" type="text" placeholder="https://example.com/image.jpg"
                        style="width: 100%; padding: 8px 12px; background: rgba(255,255,255,0.08); border: 1px solid rgba(255,255,255,0.15); border-radius: 6px; color: white; font-size: 14px;">
                </div>
                <div style="margin-bottom: 16px;">
                    <label style="color: rgba(255,255,255,0.6); font-size: 12px; display: block; margin-bottom: 4px;">Or upload file</label>
                    <input id="imageFileInput" type="file" accept="image/*"
                        style="width: 100%; padding: 8px; background: rgba(255,255,255,0.08); border: 1px solid rgba(255,255,255,0.15); border-radius: 6px; color: white; font-size: 14px;">
                </div>
                <div style="display: flex; gap: 8px; justify-content: flex-end;">
                    <button id="imageCancelBtn" style="padding: 8px 16px; background: transparent; border: 1px solid rgba(255,255,255,0.2); border-radius: 6px; color: white; cursor: pointer;">Cancel</button>
                    <button id="imageInsertBtn" style="padding: 8px 16px; background: var(--accent, #00ffcc); border: none; border-radius: 6px; color: #000; cursor: pointer; font-weight: bold;">Insert</button>
                </div>
            </div>
        `;
        document.body.appendChild(modal);

        // Handle insert
        const insertImage = () => {
            const url = document.getElementById('imageUrlInput').value.trim();
            const file = document.getElementById('imageFileInput').files[0];

            const createImg = (src) => {
                const currentSlide = this.getCurrentSlide();
                const content = currentSlide.querySelector('.slide-content');
                const img = document.createElement('img');
                img.className = 'editable-element slide-image';
                img.src = src;
                img.alt = 'Added image';
                img.style.maxHeight = 'min(50vh, 400px)';
                content.appendChild(img);
                modal.remove();
            };

            if (url) {
                createImg(url);
            } else if (file) {
                const reader = new FileReader();
                reader.onload = (e) => createImg(e.target.result);
                reader.readAsDataURL(file);
            }
        };

        document.getElementById('imageInsertBtn').addEventListener('click', insertImage);
        document.getElementById('imageCancelBtn').addEventListener('click', () => modal.remove());
        modal.addEventListener('click', (e) => { if (e.target === modal) modal.remove(); });
    });
}
```

---

## Text Insert

Add new text boxes to the current slide.

```javascript
initTextInsert() {
    const addTextBtn = document.getElementById('addTextBtn');
    addTextBtn.addEventListener('click', () => {
        const currentSlide = this.getCurrentSlide();
        const content = currentSlide.querySelector('.slide-content');
        const text = document.createElement('p');
        text.className = 'editable-element reveal visible';
        text.textContent = 'New text — click to edit';
        text.style.padding = '8px';
        text.style.margin = '8px 0';
        content.appendChild(text);

        // Focus the new text element for immediate editing
        if (this.isActive) {
            text.setAttribute('contenteditable', 'true');
            text.focus();
            // Select all text
            const range = document.createRange();
            range.selectNodeContents(text);
            const sel = window.getSelection();
            sel.removeAllRanges();
            sel.addRange(range);
        }
    });
}
```

---

## Save/Export

Ctrl+S downloads a clean HTML file with all edit state stripped.

**CRITICAL: The saved file must NOT contain any edit-mode UI or classes.**

```javascript
initSave() {
    document.addEventListener('keydown', (e) => {
        if (e.ctrlKey && e.key === 's') {
            e.preventDefault();
            this.exportFile();
        }
    });
    document.getElementById('saveBtn')?.addEventListener('click', () => this.exportFile());
}

exportFile() {
    // 1. Strip edit state
    const editableEls = Array.from(document.querySelectorAll('[contenteditable]'));
    editableEls.forEach(el => el.removeAttribute('contenteditable'));

    document.body.classList.remove('edit-active');
    document.getElementById('editToggle')?.classList.remove('active', 'show');
    document.querySelectorAll('.selected').forEach(el => el.classList.remove('selected'));

    // 2. Remove dynamically-added edit handles (critical — these must NOT appear in saved file)
    document.querySelectorAll('.drag-handle, .resize-handle').forEach(el => el.remove());

    // 3. Restore clean presentation state
    document.documentElement.style.scrollSnapType = '';
    document.querySelectorAll('.slide').forEach(s => { s.style.marginLeft = ''; });

    // 4. Capture HTML
    const html = '<!DOCTYPE html>\n' + document.documentElement.outerHTML;

    // 5. Restore edit state if still in edit mode
    if (this.isActive) {
        document.body.classList.add('edit-active');
        editableEls.forEach(el => el.setAttribute('contenteditable', 'true'));
        document.getElementById('editToggle')?.classList.add('active');
        document.querySelectorAll('.slide').forEach(s => { s.style.marginLeft = '200px'; });
    }

    // 6. Download
    const blob = new Blob([html], { type: 'text/html' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'presentation.html';
    a.click();
    URL.revokeObjectURL(a.href);
}
```

---

## Auto-save to localStorage

Persists edits automatically. Restores on page reload.

```javascript
initAutoSave() {
    // Auto-save every 10 seconds when in edit mode (debounced)
    this.autoSaveInterval = setInterval(() => {
        if (!this.isActive) return;
        const slides = document.querySelectorAll('.slide');
        const data = [];
        slides.forEach(slide => data.push(slide.outerHTML));
        try { localStorage.setItem('editable-slides-data', JSON.stringify(data)); } catch(e) {}
    }, 10000);

    // Restore on load
    const saved = localStorage.getItem('editable-slides-data');
    if (saved) {
        try {
            const data = JSON.parse(saved);
            const container = document.body;
            // Remove existing slides
            document.querySelectorAll('.slide').forEach(s => s.remove());
            // Restore saved slides
            data.forEach((html, i) => {
                const temp = document.createElement('div');
                temp.innerHTML = html;
                const slide = temp.firstElementChild;
                slide.setAttribute('data-slide-index', i);
                container.appendChild(slide);
            });
        } catch (e) {
            console.warn('Failed to restore saved data:', e);
        }
    }
}
```

---

## Complete Implementation

Here is the complete `SlideEditor` class that ties everything together. Include this in the generated HTML's `<script>` section.

```javascript
class SlideEditor {
    constructor(presentation) {
        this.presentation = presentation;
        this.isActive = false;
        this.selectedElement = null;
        this.currentSlideIndex = 0;

        this.initToggle();
        this.initToolbar();
        this.initSelection();
        this.initDrag();
        this.initResize();
        this.initImageInsert();
        this.initTextInsert();
        this.initSave();
        this.initAutoSave();
        this.initSlideTracking();
    }

    /* === MODE TOGGLE === */

    toggleEditMode() {
        this.isActive = !this.isActive;
        document.body.classList.toggle('edit-active', this.isActive);

        const editToggle = document.getElementById('editToggle');
        editToggle.classList.toggle('active', this.isActive);

        if (this.isActive) {
            // Enter edit mode
            document.documentElement.style.scrollSnapType = 'none';

            // Make text elements editable
            document.querySelectorAll('.editable-element').forEach(el => {
                if (el.tagName !== 'IMG') {
                    el.setAttribute('contenteditable', 'true');
                }
            });

            // Hide presentation-only elements
            document.querySelector('.progress-bar')?.style.setProperty('display', 'none');
            document.querySelector('.nav-dots')?.style.setProperty('display', 'none');

            // Build slide panel
            this.buildSlidePanel();

            // Shift slides right to make room for panel
            document.querySelectorAll('.slide').forEach(s => {
                s.style.marginLeft = '200px';
            });

        } else {
            // Exit edit mode — presentation mode
            document.documentElement.style.scrollSnapType = 'y mandatory';

            // Remove contenteditable
            document.querySelectorAll('[contenteditable]').forEach(el => {
                el.removeAttribute('contenteditable');
            });

            // Deselect
            this.selectElement(null);

            // Show presentation elements
            document.querySelector('.progress-bar')?.style.removeProperty('display');
            document.querySelector('.nav-dots')?.style.removeProperty('display');

            // Restore slide positions
            document.querySelectorAll('.slide').forEach(s => {
                s.style.marginLeft = '';
            });
        }
    }

    /* === ELEMENT SELECTION === */

    selectElement(el) {
        // Deselect previous
        if (this.selectedElement) {
            this.selectedElement.classList.remove('selected');
            // Remove drag handle
            const oldHandle = this.selectedElement.querySelector('.drag-handle');
            oldHandle?.remove();
            const oldResizeHandles = this.selectedElement.parentElement?.querySelectorAll('.resize-handle');
            oldResizeHandles?.forEach(h => h.remove());
        }

        this.selectedElement = el;

        if (el) {
            el.classList.add('selected');

            // Add drag handle
            if (!el.querySelector('.drag-handle')) {
                const handle = document.createElement('div');
                handle.className = 'drag-handle';
                el.appendChild(handle);
            }

            // Add resize handles for images
            if (el.tagName === 'IMG') {
                ['nw', 'ne', 'sw', 'se'].forEach(pos => {
                    if (!el.parentElement.querySelector(`.resize-handle.${pos}`)) {
                        const rh = document.createElement('div');
                        rh.className = `resize-handle ${pos}`;
                        el.parentElement.appendChild(rh);
                    }
                });
            }

            // Don't select text when clicking (only on explicit double-click)
            if (el.getAttribute('contenteditable') === 'true') {
                el.classList.add('editing');
                el.addEventListener('blur', () => el.classList.remove('editing'), { once: true });
            }
        }
    }

    /* === CLICK TO SELECT === */

    initSelection() {
        document.addEventListener('click', (e) => {
            if (!this.isActive) return;

            // Ignore toolbar, slide panel, and modals
            if (e.target.closest('.edit-toolbar') ||
                e.target.closest('.slide-panel') ||
                e.target.closest('.resize-handle') ||
                e.target.closest('.drag-handle')) return;

            const el = e.target.closest('.editable-element');
            if (el) {
                this.selectElement(el);
            } else {
                this.selectElement(null);
            }
        });
    }

    /* === SLIDE TRACKING === */

    initSlideTracking() {
        // Track which slide is currently in view
        const observer = new IntersectionObserver((entries) => {
            entries.forEach(entry => {
                if (entry.isIntersecting && entry.intersectionRatio > 0.5) {
                    const index = parseInt(entry.target.dataset.slideIndex);
                    if (!isNaN(index)) this.currentSlideIndex = index;
                    if (this.isActive) this.buildSlidePanel();
                }
            });
        }, { threshold: 0.5 });

        document.querySelectorAll('.slide').forEach(slide => observer.observe(slide));
    }

    getCurrentSlide() {
        return document.querySelectorAll('.slide')[this.currentSlideIndex];
    }

    /* === INIT METHODS === */

    initToggle() {
        const hotzone = document.querySelector('.edit-hotzone');
        const editToggle = document.getElementById('editToggle');
        if (!hotzone || !editToggle) return;
        let hideTimeout = null;

        hotzone.addEventListener('mouseenter', () => {
            clearTimeout(hideTimeout);
            editToggle.classList.add('show');
        });
        hotzone.addEventListener('mouseleave', () => {
            hideTimeout = setTimeout(() => {
                if (!this.isActive) editToggle.classList.remove('show');
            }, 400);
        });
        editToggle.addEventListener('mouseenter', () => clearTimeout(hideTimeout));
        editToggle.addEventListener('mouseleave', () => {
            hideTimeout = setTimeout(() => {
                if (!this.isActive) editToggle.classList.remove('show');
            }, 400);
        });
        editToggle.addEventListener('click', () => this.toggleEditMode());
        hotzone.addEventListener('click', () => this.toggleEditMode());
        document.addEventListener('keydown', (e) => {
            if ((e.key === 'e' || e.key === 'E') && !e.target.getAttribute('contenteditable')) {
                e.preventDefault();
                this.toggleEditMode();
            }
        });
    }

    initToolbar() {
        // Formatting commands
        document.querySelectorAll('.toolbar-btn[data-command]').forEach(btn => {
            btn.addEventListener('click', (e) => {
                e.preventDefault();
                document.execCommand(btn.dataset.command, false, null);
                this.updateToolbarState();
            });
        });
        // Font size
        const fontSizeSelect = document.getElementById('fontSizeSelect');
        fontSizeSelect?.addEventListener('change', (e) => {
            document.execCommand('fontSize', false, e.target.value);
        });
        // Text color
        const colorPicker = document.getElementById('textColorPicker');
        colorPicker?.addEventListener('input', (e) => {
            document.execCommand('foreColor', false, e.target.value);
        });
        // Selection state tracking
        document.addEventListener('selectionchange', () => this.updateToolbarState());
    }

    updateToolbarState() {
        document.querySelectorAll('.toolbar-btn[data-command]').forEach(btn => {
            const cmd = btn.dataset.command;
            if (['bold', 'italic', 'underline', 'justifyLeft', 'justifyCenter',
                 'justifyRight', 'insertUnorderedList', 'insertOrderedList'].includes(cmd)) {
                btn.classList.toggle('active', document.queryCommandState(cmd));
            }
        });
    }

    initDrag() {
        let isDragging = false;
        let currentEl = null;
        let startX, startY, startLeft, startTop;

        // Mouse events
        document.addEventListener('mousedown', (e) => {
            if (!this.isActive) return;
            const handle = e.target.closest('.drag-handle');
            if (!handle) return;
            e.preventDefault();
            this._startDrag(handle.parentElement, e.clientX, e.clientY);
            isDragging = true;
            currentEl = handle.parentElement;
            startX = e.clientX; startY = e.clientY;
            const slideContent = currentEl.closest('.slide-content');
            const slideRect = slideContent.getBoundingClientRect();
            startLeft = parseInt(currentEl.style.left) || 0;
            startTop = parseInt(currentEl.style.top) || 0;
            // Store slideRect for delta calculations
            this._dragSlideRect = slideRect;
            this._dragStartLeft = startLeft;
            this._dragStartTop = startTop;
        });

        document.addEventListener('mousemove', (e) => {
            if (!isDragging || !currentEl) return;
            e.preventDefault();
            const dx = e.clientX - startX;
            const dy = e.clientY - startY;
            currentEl.style.left = (this._dragStartLeft + dx) + 'px';
            currentEl.style.top = (this._dragStartTop + dy) + 'px';
        });

        document.addEventListener('mouseup', () => {
            if (currentEl) {
                currentEl.style.zIndex = '';
                currentEl.style.opacity = '';
            }
            isDragging = false; currentEl = null;
        });

        // Touch events
        document.addEventListener('touchstart', (e) => {
            if (!this.isActive) return;
            const handle = e.target.closest('.drag-handle');
            if (!handle) return;
            const touch = e.touches[0];
            isDragging = true; currentEl = handle.parentElement;
            startX = touch.clientX; startY = touch.clientY;
            const slideContent = currentEl.closest('.slide-content');
            const slideRect = slideContent.getBoundingClientRect();
            if (!currentEl.style.position || currentEl.style.position !== 'absolute') {
                const elRect = currentEl.getBoundingClientRect();
                currentEl.style.position = 'absolute';
                currentEl.style.left = (elRect.left - slideRect.left) + 'px';
                currentEl.style.top = (elRect.top - slideRect.top) + 'px';
                currentEl.style.margin = '0';
            }
            startLeft = parseInt(currentEl.style.left);
            startTop = parseInt(currentEl.style.top);
            this._dragStartLeft = startLeft; this._dragStartTop = startTop;
        }, { passive: true });

        document.addEventListener('touchmove', (e) => {
            if (!isDragging || !currentEl) return;
            e.preventDefault();
            const touch = e.touches[0];
            const dx = touch.clientX - startX;
            const dy = touch.clientY - startY;
            currentEl.style.left = (this._dragStartLeft + dx) + 'px';
            currentEl.style.top = (this._dragStartTop + dy) + 'px';
        }, { passive: false });

        document.addEventListener('touchend', () => {
            if (currentEl) { currentEl.style.zIndex = ''; currentEl.style.opacity = ''; }
            isDragging = false; currentEl = null;
        });
    }

    _startDrag(el, clientX, clientY) {
        const slideContent = el.closest('.slide-content');
        if (!slideContent) return;
        const slideRect = slideContent.getBoundingClientRect();
        if (!el.style.position || el.style.position !== 'absolute') {
            const elRect = el.getBoundingClientRect();
            el.style.position = 'absolute';
            el.style.left = (elRect.left - slideRect.left) + 'px';
            el.style.top = (elRect.top - slideRect.top) + 'px';
            el.style.margin = '0';
        }
        el.style.zIndex = '100';
        el.style.opacity = '0.8';
    }

    initResize() {
        let isResizing = false, currentImg = null;
        let startX, startY, startW, startH, aspectRatio = 1;

        const onStart = (img, clientX, clientY) => {
            isResizing = true; currentImg = img;
            aspectRatio = img.naturalWidth / img.naturalHeight;
            startX = clientX; startY = clientY;
            startW = img.offsetWidth; startH = img.offsetHeight;
        };
        const onMove = (clientX, clientY, maintainAspect) => {
            if (!isResizing || !currentImg) return;
            const dx = clientX - startX, dy = clientY - startY;
            let newW, newH;
            if (maintainAspect) {
                newW = Math.max(50, startW + dx);
                newH = newW / aspectRatio;
            } else {
                newW = Math.max(50, startW + dx);
                newH = Math.max(50, startH + dy);
            }
            currentImg.style.width = newW + 'px';
            currentImg.style.height = newH + 'px';
            currentImg.style.maxWidth = 'none';
            currentImg.style.maxHeight = 'none';
        };
        const onEnd = () => {
            if (isResizing) currentImg?.classList.remove('selected');
            isResizing = false; currentImg = null;
        };

        // Mouse
        document.addEventListener('mousedown', (e) => {
            if (!this.isActive) return;
            const handle = e.target.closest('.resize-handle');
            if (!handle) return;
            e.preventDefault();
            const img = handle.closest('img.editable-element') || handle.previousElementSibling;
            if (!img || img.tagName !== 'IMG') return;
            onStart(img, e.clientX, e.clientY);
        });
        document.addEventListener('mousemove', (e) => {
            if (!isResizing) return;
            e.preventDefault();
            onMove(e.clientX, e.clientY, !e.shiftKey);
        });
        document.addEventListener('mouseup', onEnd);

        // Touch
        document.addEventListener('touchstart', (e) => {
            if (!this.isActive) return;
            const handle = e.target.closest('.resize-handle');
            if (!handle) return;
            const img = handle.closest('img.editable-element') || handle.previousElementSibling;
            if (!img || img.tagName !== 'IMG') return;
            onStart(img, e.touches[0].clientX, e.touches[0].clientY);
        }, { passive: true });
        document.addEventListener('touchmove', (e) => {
            if (!isResizing) return;
            e.preventDefault();
            onMove(e.touches[0].clientX, e.touches[0].clientY, true);
        }, { passive: false });
        document.addEventListener('touchend', onEnd);
    }

    initImageInsert() {
        const btn = document.getElementById('addImageBtn');
        btn?.addEventListener('click', () => {
            const modal = document.createElement('div');
            modal.style.cssText = 'position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(0,0,0,0.7);z-index:10002;display:flex;align-items:center;justify-content:center;';
            modal.innerHTML = `
                <div style="background:#1e1e28;border-radius:12px;padding:24px;min-width:400px;max-width:500px;">
                    <h3 style="color:white;margin-bottom:16px;">Add Image</h3>
                    <div style="margin-bottom:12px;">
                        <label style="color:rgba(255,255,255,0.6);font-size:12px;display:block;margin-bottom:4px;">From URL</label>
                        <input id="imageUrlInput" type="text" placeholder="https://example.com/image.jpg"
                            style="width:100%;padding:8px 12px;background:rgba(255,255,255,0.08);border:1px solid rgba(255,255,255,0.15);border-radius:6px;color:white;font-size:14px;">
                    </div>
                    <div style="margin-bottom:16px;">
                        <label style="color:rgba(255,255,255,0.6);font-size:12px;display:block;margin-bottom:4px;">Or upload file</label>
                        <input id="imageFileInput" type="file" accept="image/*"
                            style="width:100%;padding:8px;background:rgba(255,255,255,0.08);border:1px solid rgba(255,255,255,0.15);border-radius:6px;color:white;font-size:14px;">
                    </div>
                    <div style="display:flex;gap:8px;justify-content:flex-end;">
                        <button id="imageCancelBtn" style="padding:8px 16px;background:transparent;border:1px solid rgba(255,255,255,0.2);border-radius:6px;color:white;cursor:pointer;">Cancel</button>
                        <button id="imageInsertBtn" style="padding:8px 16px;background:var(--accent,#00ffcc);border:none;border-radius:6px;color:#000;cursor:pointer;font-weight:bold;">Insert</button>
                    </div>
                </div>`;
            document.body.appendChild(modal);
            const insertImage = () => {
                const url = document.getElementById('imageUrlInput').value.trim();
                const file = document.getElementById('imageFileInput').files[0];
                const createImg = (src) => {
                    const content = this.getCurrentSlide()?.querySelector('.slide-content');
                    if (!content) { modal.remove(); return; }
                    const img = document.createElement('img');
                    img.className = 'editable-element slide-image';
                    img.src = src; img.alt = 'Added image';
                    img.style.maxHeight = 'min(50vh, 400px)';
                    content.appendChild(img);
                    modal.remove();
                };
                if (url) createImg(url);
                else if (file) {
                    const reader = new FileReader();
                    reader.onload = (e) => createImg(e.target.result);
                    reader.readAsDataURL(file);
                }
            };
            document.getElementById('imageInsertBtn').addEventListener('click', insertImage);
            document.getElementById('imageCancelBtn').addEventListener('click', () => modal.remove());
            modal.addEventListener('click', (e) => { if (e.target === modal) modal.remove(); });
        });
    }

    initTextInsert() {
        const btn = document.getElementById('addTextBtn');
        btn?.addEventListener('click', () => {
            const content = this.getCurrentSlide()?.querySelector('.slide-content');
            if (!content) return;
            const text = document.createElement('p');
            text.className = 'editable-element reveal visible';
            text.textContent = 'New text — click to edit';
            text.style.padding = '8px'; text.style.margin = '8px 0';
            content.appendChild(text);
            if (this.isActive) {
                text.setAttribute('contenteditable', 'true');
                text.focus();
                const range = document.createRange();
                range.selectNodeContents(text);
                const sel = window.getSelection();
                sel.removeAllRanges(); sel.addRange(range);
            }
        });
    }

    initSave() {
        document.addEventListener('keydown', (e) => {
            if (e.ctrlKey && e.key === 's') { e.preventDefault(); this.exportFile(); }
        });
        document.getElementById('saveBtn')?.addEventListener('click', () => this.exportFile());
        document.getElementById('presentBtn')?.addEventListener('click', () => {
            if (this.isActive) this.toggleEditMode();
        });
    }

    initAutoSave() {
        this.autoSaveInterval = setInterval(() => {
            if (!this.isActive) return;
            const slides = document.querySelectorAll('.slide');
            const data = [];
            slides.forEach(slide => data.push(slide.outerHTML));
            try { localStorage.setItem('editable-slides-data', JSON.stringify(data)); } catch(e) {}
        }, 10000);

        // Restore on load — insert slides before the toolbar to maintain DOM order
        const saved = localStorage.getItem('editable-slides-data');
        if (saved) {
            try {
                const data = JSON.parse(saved);
                document.querySelectorAll('.slide').forEach(s => s.remove());
                const toolbar = document.getElementById('editToolbar');
                data.forEach((html, i) => {
                    const temp = document.createElement('div');
                    temp.innerHTML = html;
                    const slide = temp.firstElementChild;
                    slide.setAttribute('data-slide-index', i);
                    // Insert before toolbar to keep DOM order correct
                    if (toolbar) document.body.insertBefore(slide, toolbar);
                    else document.body.appendChild(slide);
                });
            } catch (e) { console.warn('Failed to restore saved data:', e); }
        }
    }

    buildSlidePanel() {
        const panel = document.getElementById('slidePanel');
        if (!panel) return;
        panel.innerHTML = '';
        const slides = document.querySelectorAll('.slide');
        slides.forEach((slide, i) => {
            const thumb = document.createElement('div');
            thumb.className = 'slide-thumb' + (i === this.currentSlideIndex ? ' active' : '');
            thumb.innerHTML = `<span class="slide-thumb-label">${i + 1}</span>
                <div class="slide-thumb-actions">
                    <button class="slide-thumb-btn duplicate" data-action="duplicate" data-index="${i}" title="Duplicate" aria-label="Duplicate slide ${i+1}">⧉</button>
                    <button class="slide-thumb-btn" data-action="delete" data-index="${i}" title="Delete" aria-label="Delete slide ${i+1}">✕</button>
                </div>`;
            thumb.addEventListener('click', (e) => {
                if (e.target.closest('.slide-thumb-btn')) return;
                this.scrollToSlide(i);
            });
            panel.appendChild(thumb);
        });
        const addBtn = document.createElement('div');
        addBtn.className = 'panel-add-slide'; addBtn.textContent = '+';
        addBtn.setAttribute('aria-label', 'Add new slide');
        addBtn.addEventListener('click', () => this.addSlide());
        panel.appendChild(addBtn);
        panel.querySelectorAll('.slide-thumb-btn').forEach(btn => {
            btn.addEventListener('click', (e) => {
                e.stopPropagation();
                const index = parseInt(btn.dataset.index);
                if (btn.dataset.action === 'delete') this.deleteSlide(index);
                if (btn.dataset.action === 'duplicate') this.duplicateSlide(index);
            });
        });
    }

    scrollToSlide(index) {
        const slides = document.querySelectorAll('.slide');
        if (index >= 0 && index < slides.length) {
            slides[index].scrollIntoView({ behavior: 'smooth' });
            this.currentSlideIndex = index;
            this.buildSlidePanel();
        }
    }

    addSlide() {
        const slides = document.querySelectorAll('.slide');
        const newSlide = document.createElement('section');
        newSlide.className = 'slide';
        newSlide.setAttribute('data-slide-index', slides.length);
        newSlide.innerHTML = `<div class="slide-content">
            <h2 class="editable-element reveal visible">New Slide</h2>
            <p class="editable-element reveal visible">Click to add content</p>
        </div>`;
        document.body.appendChild(newSlide);
        this.buildSlidePanel();
        this.scrollToSlide(slides.length);
    }

    deleteSlide(index) {
        const slides = document.querySelectorAll('.slide');
        if (slides.length <= 1) return;
        if (confirm('Delete this slide?')) {
            slides[index].remove();
            document.querySelectorAll('.slide').forEach((s, i) => s.setAttribute('data-slide-index', i));
            if (this.currentSlideIndex >= document.querySelectorAll('.slide').length) {
                this.currentSlideIndex = document.querySelectorAll('.slide').length - 1;
            }
            this.buildSlidePanel();
        }
    }

    duplicateSlide(index) {
        const slides = document.querySelectorAll('.slide');
        const clone = slides[index].cloneNode(true);
        clone.querySelectorAll('.selected').forEach(el => el.classList.remove('selected'));
        clone.querySelectorAll('[contenteditable]').forEach(el => el.removeAttribute('contenteditable'));
        document.body.insertBefore(clone, slides[index].nextSibling);
        document.querySelectorAll('.slide').forEach((s, i) => s.setAttribute('data-slide-index', i));
        this.buildSlidePanel();
    }

    exportFile() {
        // 1. Strip edit state
        const editableEls = Array.from(document.querySelectorAll('[contenteditable]'));
        editableEls.forEach(el => el.removeAttribute('contenteditable'));
        document.body.classList.remove('edit-active');
        document.getElementById('editToggle')?.classList.remove('active', 'show');
        document.querySelectorAll('.selected').forEach(el => el.classList.remove('selected'));
        document.querySelectorAll('.drag-handle, .resize-handle').forEach(el => el.remove());
        document.documentElement.style.scrollSnapType = '';
        document.querySelectorAll('.slide').forEach(s => { s.style.marginLeft = ''; });

        // 2. Capture HTML
        const html = '<!DOCTYPE html>\n' + document.documentElement.outerHTML;

        // 3. Restore edit state
        if (this.isActive) {
            document.body.classList.add('edit-active');
            editableEls.forEach(el => el.setAttribute('contenteditable', 'true'));
            document.getElementById('editToggle')?.classList.add('active');
            if (this.selectedElement) {
                this.selectedElement.classList.add('selected');
                if (!this.selectedElement.querySelector('.drag-handle')) {
                    const h = document.createElement('div');
                    h.className = 'drag-handle';
                    this.selectedElement.appendChild(h);
                }
            }
            document.querySelectorAll('.slide').forEach(s => { s.style.marginLeft = '200px'; });
        }

        // 4. Download
        const blob = new Blob([html], { type: 'text/html' });
        const a = document.createElement('a');
        a.href = URL.createObjectURL(blob);
        a.download = 'presentation.html';
        a.click();
        URL.revokeObjectURL(a.href);
    }
}

// Initialize after DOM is ready
document.addEventListener('DOMContentLoaded', () => {
    const presentation = new SlidePresentation();
    const editor = new SlideEditor(presentation);
});
```

---

## Important Notes for Generation

1. **Always include the complete SlideEditor class.** Do not truncate or abbreviate it — the editor must work fully in the generated HTML.

2. **Mark all user-visible text with `class="editable-element"`.** This includes headings, paragraphs, bullet items, quotes, captions, and any other text content.

3. **Mark all images with `class="editable-element slide-image"`.**

4. **The toolbar must use simple Unicode/symbols, not icon libraries** — keeping it zero-dependency.

5. **Test that exportFile() strips ALL edit state** — the saved file should open cleanly in presentation mode.

6. **The slide-panel width (200px) must match the slide margin-left shift** when entering edit mode.

7. **contenteditable is added/removed by JS at toggle time** — never hardcode it in the HTML.
