# Vellum to Vertical CJK EPUB: The Complete Guide to Converting English-Layout EPUBs for Traditional Chinese, Japanese, and Korean Vertical Text

> **A step-by-step guide to converting Vellum-exported EPUBs into properly formatted vertical right-to-left CJK e-books for Apple Books, Kobo, and other EPUB readers.**

[Vellum](https://vellum.pub/) is one of the most popular tools for formatting and exporting e-books on macOS. It produces beautifully structured EPUB and Kindle files — but it only supports horizontal left-to-right text. If you're publishing a book in Traditional Chinese (繁體中文), Japanese, or Korean that needs vertical text layout (`writing-mode: vertical-rl`), Vellum has no built-in option for this.

This guide documents a complete, working approach to taking a Vellum-generated EPUB and converting it to vertical CJK layout through targeted CSS overrides and XHTML modifications. Every technique here was developed through real-world trial and error while formatting a Traditional Chinese book for Apple Books and Kobo, and all output was validated with EPUBCheck.

**What this guide covers:**

- Setting up `writing-mode: vertical-rl` globally without breaking Vellum's class system
- Understanding how CSS axes remap in vertical writing (the #1 source of bugs)
- Centering content on pages (dedication, epigraph, part dividers) — what works and what doesn't
- Fixing headings, subheadings, and paragraph spacing for vertical text
- Cross-device compatibility between Apple Books and Kobo (WebKit quirks, logical properties)
- Adding new front matter pages (dedication, epigraph) to Vellum's structure
- Table of contents hierarchy adjustments
- EPUB packaging and validation

**What you'll need:**

- A Vellum-exported EPUB file (the starting point)
- A text editor for CSS and XHTML modifications
- Python `epubcheck` package (or standalone EPUBCheck) for validation
- Standard `zip`/`unzip` for EPUB repackaging

---

## Background: Why Vellum + Vertical CJK Is Hard

Vellum produces clean, well-structured EPUBs, but its CSS and HTML are built entirely around horizontal left-to-right text. Converting to vertical right-to-left CJK layout requires overriding many of Vellum's assumptions about spacing, alignment, and centering — without breaking its carefully constructed class system.

The fundamental challenge is that switching `writing-mode` from `horizontal-tb` to `vertical-rl` remaps every CSS axis. Properties like `margin-top`, `text-align: left`, and `padding-top` all change meaning. Vellum's heading system, paragraph spacing, and page centering all assume horizontal layout and must be systematically overridden.

**Key principle:** Never remove Vellum's original CSS. Instead, append override rules at the end of `style.css` using `!important` where needed. This preserves the existing structure while adapting it for vertical text.

---

## 1. Setting the Global Writing Mode

Add to the end of `style.css`:

```css
html {
    writing-mode: tb-rl;           /* Legacy fallback */
    writing-mode: vertical-rl;     /* Standard */
    -epub-writing-mode: vertical-rl; /* EPUB spec */
    -webkit-writing-mode: vertical-rl; /* Apple Books / Kobo WebKit */
}

body {
    margin: 0;
    padding: 0;
}
```

**What changes:** Text now flows top-to-bottom in columns, columns flow right-to-left. Every CSS property that depends on direction is affected — `text-align: left` now means "top of column," `margin-top` becomes the inline-start margin, and `margin-right` becomes the block-start margin.

---

## 2. Understanding Axis Mapping in Vertical-RL

This is the single most important concept. Almost every bug we encountered came from forgetting how axes remap.

| Concept | Horizontal-TB | Vertical-RL |
|---|---|---|
| Inline direction | Left → Right | Top → Bottom |
| Block direction | Top → Bottom | Right → Left |
| `text-align: left` | Aligns to left edge | Aligns to **top** of column |
| `text-align: right` | Aligns to right edge | Aligns to **bottom** of column |
| `margin-top` (physical) | Space above element | **Inline-start** space (pushes text down from top) |
| `margin-right` (physical) | Space to the right | **Block-start** space (space before the column) |
| `padding-top` (physical) | Constrains column **height** | Constrains column **height** |
| `padding-left/right` (physical) | Left/right page margins | Space where **columns stack** |

**Critical implication:** When you add `padding-top` to a container, you're reducing the height available for each text column. This forces text into more (shorter) columns, making the block wider. This is counterintuitive and was the source of several failed centering attempts.

---

## 3. Centering Content on a Page

This was the hardest problem to solve. Here's what works and what doesn't.

### What works: Absolute positioning with transform

This technique centers content at the physical center of the viewport regardless of writing mode:

```html
<body style="margin: 0; padding: 0;">
  <div style="position: absolute; top: 0; left: 0;
              width: 100vw; height: 100vh;
              margin: 0; padding: 0; overflow: hidden;">
    <div style="position: absolute; top: 50%; left: 50%;
                transform: translate(-50%, -50%);
                -webkit-transform: translate(-50%, -50%);">
      <!-- Your content here -->
    </div>
  </div>
</body>
```

**Why it works:** `top`, `left`, and `transform` operate in physical coordinates, completely independent of writing mode. The outer div creates a viewport-sized positioning context; the inner div is placed at the exact center.

**Used for:** Dedication pages, epigraph pages, part/section divider pages.

**Important:** For the inner content div, you may need to set an explicit `height` (e.g., `height: 90vh`) to give the text columns enough vertical space. Without this, columns may be too short or unpredictably sized.

### What doesn't work

- **Flexbox (`display: flex; align-items: center; justify-content: center`):** In vertical-rl, the flex axes remap. What centers horizontally in horizontal-tb ends up misaligned in vertical-rl. Text appeared in the bottom-right corner.

- **Table-cell centering (`display: table-cell; vertical-align: middle`):** Only works if you force `writing-mode: horizontal-tb` on the body, which displays text horizontally — defeating the purpose for CJK vertical text.

- **Vellum's heading classes (`heading-container-group`, `heading-alignment-fixed`):** These use padding and min-height to position content, which pushes text down from the top rather than centering it. In vertical-rl, this translates to unpredictable offsets.

---

## 4. Section Headings

Vellum's heading system uses `min-height`, `padding-top`, and centered alignment to create visual space before chapter titles. In vertical-rl, these properties push the title away from the top of the column, creating unwanted indentation.

**Fix:** Zero out all min-height and padding, force `text-align: left` (which means "top of column" in vertical-rl):

```css
.element-type-chapter .heading-container-single,
.element-type-chapter .heading-contents,
.element-type-chapter .title-block,
.element-type-chapter .element-title {
    min-height: 0 !important;
    height: auto !important;
    padding-top: 0 !important;
    margin-top: 0 !important;
    text-align: left !important;
}
```

Also zero out the 6% left/right margins on `heading-container-single` — in vertical-rl these become block-direction margins and can push body text to the next page, creating blank pages:

```css
.element-type-chapter .heading-container-single {
    margin-left: 0 !important;
    margin-right: 0 !important;
}
```

Apply the same pattern to all element types: `introduction`, `bibliography`, `acknowledgments`, `about-author`, `epilogue`, `toc`.

---

## 5. Subheading Spacing

Vellum subheadings (`h2.section-title.subhead`) need both physical property fallbacks (for Kobo's older WebKit) and CSS logical properties (for Apple Books):

```css
h2.section-title.subhead.level-1 {
    /* Zero physical top/bottom — these become inline-start/end in vertical-rl */
    margin-top: 0 !important;
    margin-bottom: 0 !important;
    padding-top: 0 !important;
    padding-bottom: 0 !important;

    /* Physical fallback for Kobo (margin-right = block-start in vertical-rl) */
    margin-right: 2.5em !important;
    margin-left: 0 !important;

    /* Logical properties for modern engines (Apple Books) */
    margin-block-start: 2.5em !important;
    margin-block-end: 0 !important;
}
```

**Why both?** Kobo's rendering engine doesn't fully support CSS logical properties. By setting the physical `margin-right` (which equals block-start in vertical-rl) alongside `margin-block-start`, you get correct spacing on both platforms.

---

## 6. Paragraph Spacing and Indentation

### Remove inter-paragraph gaps

In vertical CJK text, paragraphs are signaled by indentation, not by vertical gaps. Zero out all paragraph margins:

```css
p, p.subsq, p.first, p.first-in-chapter,
p.first-full-width, p.first-after-subhead {
    margin: 0 !important;
    padding: 0 !important;
    -webkit-margin-before: 0 !important;
    -webkit-margin-after: 0 !important;
    -webkit-margin-start: 0 !important;
    -webkit-margin-end: 0 !important;
    margin-block-start: 0 !important;
    margin-block-end: 0 !important;
    border: none !important;
}
```

The `-webkit-margin-*` properties are necessary because Apple Books and Kobo internally set paragraph margins using these WebKit-specific properties, and standard `margin: 0` alone doesn't override them.

### Indent first paragraphs

Vellum's convention is to not indent the first paragraph of a section (Western typographic tradition). For CJK, all paragraphs should be indented:

```css
p.first-in-chapter,
p.first-full-width,
p.first-after-subhead,
p.first-after-blockquote {
    text-indent: 1.5em !important;
}
.text > p.first {
    text-indent: 1.5em !important;
}
```

The `.text > p.first` selector targets first paragraphs within body text containers without affecting TOC, copyright, or other structural elements.

---

## 7. Part/Section Divider Pages

Full-screen colored background pages with centered titles. The technique combines viewport coverage with absolute centering:

```css
#upart-1, #upart-2, #upart-3 {
    background-color: #636463 !important;
    color: #ffffff !important;
    position: absolute !important;
    top: 0 !important;
    left: 0 !important;
    width: 100vw !important;
    height: 100vh !important;
    margin: 0 !important;
    padding: 0 !important;
    z-index: 100 !important;
    overflow: hidden !important;
}

/* Force all child text white */
#upart-1 *, #upart-2 *, #upart-3 * {
    color: #ffffff !important;
}

/* Center heading with absolute + transform */
#upart-1 .heading, #upart-2 .heading, #upart-3 .heading {
    position: absolute !important;
    top: 50% !important;
    left: 50% !important;
    transform: translate(-50%, -50%) !important;
    -webkit-transform: translate(-50%, -50%) !important;
    margin: 0 !important;
    padding: 0 !important;
    min-height: 0 !important;
    text-align: center !important;
}
```

**Important:** Hide the empty `.text` div that Vellum generates — it can cause visual bleed-through on Kobo:

```css
#upart-1-text, #upart-2-text, #upart-3-text {
    display: none !important;
    height: 0 !important;
    overflow: hidden !important;
}
```

Also zero out all Vellum heading subcontainer dimensions (`heading-container-group`, `heading-contents`, `element-number-block`, `heading-size-full`) that add min-height or padding.

---

## 8. Cover Image

The cover page should stay horizontal so the image displays correctly:

```css
.element-type-cover, body.cover {
    margin: 0;
    padding: 0;
    overflow: hidden;
    text-align: center;
    -epub-writing-mode: horizontal-tb;
    -webkit-writing-mode: horizontal-tb;
    writing-mode: horizontal-tb;
}

img.cover-image, .element-type-cover img {
    max-width: 100%;
    max-height: 100%;
    width: auto;
    height: auto;
    display: block;
    margin: 0 auto;
    object-fit: contain;
}
```

---

## 9. Table of Contents

### In-book TOC (Vellum's visual TOC)

Vellum uses different CSS classes to create visual hierarchy: `toc-item-entry-type-part` for parts and `toc-item-entry-type-element` for front/back matter. To make prologue and epilogue entries visually equal to part entries, wrap them in the same class structure:

```html
<div class="toc-group toc-group-part">
  <div class="toc-item toc-item-entry-type-part toc-item-element-container-group
              has-no-leading-number has-no-author has-no-children">
    <p class="toc-content toc-item-title">
      <a href="introduction.xhtml">
        <span class="toc-item-title-text cspan">前言 ...</span>
      </a>
    </p>
  </div>
</div>
```

### Navigation TOC (`toc.xhtml`) and NCX (`toc.ncx`)

For flat hierarchy (all chapters/parts at the same level), place all `<li>` elements as siblings in the nav `<ol>`, and all `<navPoint>` elements at the top level in the NCX.

---

## 10. Adding New Pages (Dedication, Epigraph)

### Manifest and spine

Add entries to `content.opf`:

```xml
<!-- In <manifest> -->
<item id="dedication-xhtml" href="dedication.xhtml" media-type="application/xhtml+xml"/>
<item id="epigraph-xhtml" href="epigraph.xhtml" media-type="application/xhtml+xml"/>

<!-- In <spine>, after title-page -->
<itemref idref="dedication-xhtml"/>
<itemref idref="epigraph-xhtml"/>
```

### Dedication page example

Short text, perfectly centered:

```html
<body style="margin: 0; padding: 0;">
  <div id="dedication" epub:type="dedication"
       style="position: absolute; top: 0; left: 0;
              width: 100vw; height: 100vh;
              margin: 0; padding: 0; overflow: hidden;">
    <p style="position: absolute; top: 50%; left: 50%;
              transform: translate(-50%, -50%);
              -webkit-transform: translate(-50%, -50%);
              text-indent: 0; font-style: italic; font-size: 115%;
              margin: 0; padding: 0; white-space: nowrap;">
      獻給我的女兒
    </p>
  </div>
</body>
```

### Epigraph page example

Longer text, centered with explicit height for column sizing:

```html
<body style="margin: 0; padding: 0;">
  <div id="epigraph" epub:type="epigraph"
       style="position: absolute; top: 0; left: 0;
              width: 100vw; height: 100vh;
              margin: 0; padding: 0;">
    <div style="position: absolute; top: 50%; left: 50%;
                transform: translate(-50%, -50%);
                -webkit-transform: translate(-50%, -50%);
                height: 90vh;">
      <p style="text-indent: 0;">Quote text...</p>
      <p style="text-indent: 0; text-align: right; margin: 1.5em 0 0 0;">
        ─Attribution
      </p>
    </div>
  </div>
</body>
```

---

## 11. Title Page Adjustments

In vertical-rl, the title block (first in DOM) appears on the right side, and the author block appears on the left. Key adjustments:

- Use `font-size: 90%` on a long subtitle to keep it on fewer columns
- For the author byline character (著), use a smaller font: `<span style="font-size: 50%; font-weight: normal;"> 著</span>`
- Add `letter-spacing` to the title for visual breathing room
- Use `margin-right` on the author block to push it away from the title (since margin-right = block-start in vertical-rl)

---

## 12. Blockquote Attributions

Vellum auto-generates attribution dashes via CSS `::before` and applies `text-transform: uppercase`. Override both:

```css
p.blockquote-attribution,
p.blockquote-attribution span.case-upper,
p.blockquote-attribution span.ttext {
    text-transform: none !important;
}

p.blockquote-attribution {
    text-align: right !important; /* = bottom in vertical-rl */
}

p.blockquote-attribution::before {
    content: none !important;
}
```

Then use inline text for the dash character (─) directly in the XHTML.

---

## 13. Device-Specific Fixes

### Kobo WebKit fallback

```css
@media not all and (min-resolution: 0.001dpcm) {
    html {
        -webkit-writing-mode: vertical-rl;
    }
}
```

### Word wrapping and hyphenation

```css
p, li, blockquote {
    overflow-wrap: break-word;
    word-wrap: break-word;
}

:lang(zh), :lang(zh-TW), :lang(zh-Hant) {
    hyphens: none;
    -webkit-hyphens: none;
    -epub-hyphens: none;
}
```

---

## 14. EPUB Packaging

Always package with `mimetype` first and uncompressed:

```bash
cd your_epub_directory
zip -X0 output.epub mimetype
zip -Xr9 output.epub META-INF OEBPS
```

For Kobo, copy the file with a `.kepub.epub` extension — this triggers Kobo's enhanced rendering engine:

```bash
cp output.epub "Book Title.kepub.epub"
```

---

## 15. Validation

Always validate with EPUBCheck after changes:

```python
from epubcheck import EpubCheck
result = EpubCheck('output.epub')
print('Valid:', result.valid)
for m in result.messages:
    print(m)
```

Or via command line:

```bash
java -jar epubcheck.jar output.epub
```

---

## Common Pitfalls

1. **Flexbox centering in vertical-rl:** Axes remap and nothing lands where you expect. Use absolute + transform instead.

2. **Adding `padding-top/bottom` to "add margins":** In vertical-rl, this constrains column height, forcing text into more (shorter) columns. To add space on the top/bottom edges of a page, you need `padding-left/right` (or constrain the container dimensions).

3. **Forgetting `-webkit-` prefixes:** Apple Books and Kobo both use WebKit. Always include `-webkit-writing-mode`, `-webkit-transform`, and `-webkit-margin-*` alongside standard properties.

4. **Vellum's `min-height` on heading containers:** These create visual space in horizontal mode but translate to unwanted block-direction space in vertical-rl. Always zero them out.

5. **Assuming `margin: 0` clears all margins:** WebKit readers set internal margins via `-webkit-margin-before/after/start/end`. You must explicitly zero these.

6. **Paragraph spacing via margins:** In CJK vertical text, paragraphs are separated by indentation. Visible gaps between paragraphs look wrong. Zero all paragraph margins and rely on `text-indent` for `p.subsq`.

7. **Not testing on actual devices:** Apple Books on macOS/iOS, Kobo desktop app, and physical Kobo readers all render slightly differently. Test on all target platforms.

---

## File Structure Reference

```
OEBPS/
  content.opf          # Manifest, spine, metadata
  toc.xhtml            # EPUB 3 navigation document
  toc.ncx              # EPUB 2 fallback navigation
  cover.xhtml          # Cover image page
  title-page.xhtml     # Title page
  dedication.xhtml     # Dedication (added)
  epigraph.xhtml       # Epigraph (added)
  table-of-contents.xhtml  # Vellum's visual TOC
  introduction.xhtml   # Prologue / front matter
  upart-001.xhtml      # Part 1 divider page
  upart-001-chapter.xhtml  # Chapter 1
  ...
  css/
    style.css           # Main stylesheet (append overrides here)
    media.css           # Responsive/media queries
  fonts/                # Embedded fonts
  images/               # Cover and inline images
```

---

## Contributing

Found a better solution for any of these problems? PRs welcome. This guide was born from real formatting work on a Traditional Chinese memoir, so there are certainly edge cases and reader-specific quirks we haven't encountered yet. Japanese and Korean vertical text may have additional considerations.

## License

MIT License. Use it, adapt it, share it.

---

**Keywords:** Vellum EPUB vertical text, CJK vertical layout EPUB, Traditional Chinese e-book formatting, writing-mode vertical-rl EPUB, Apple Books vertical Chinese, Kobo vertical CJK, EPUB 3 vertical text, convert Vellum to Chinese, convert Vellum to Japanese, vertical writing CSS e-book, 繁體中文電子書, 直書排版, EPUB直排, Vellum中文
