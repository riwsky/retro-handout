---
name: ia-presenter
description: Author and iterate on iA Presenter presentations (.iapresenter bundles) on macOS. Use whenever the user mentions iA Presenter, .iapresenter, an upcoming talk or slide deck where iA Presenter is the target tool, or wants to script-first / markdown-based slides. Also use when iterating on slide layout, themes, image sizing, or when iA Presenter's import is producing wrong slide breaks. iA Presenter has no CLI or API — this skill documents the bundle structure and the hot-reload-text.md trick that's the de facto agent-friendly path, plus the small set of image/layout directives that solve 90% of layout issues.
---

# iA Presenter

iA Presenter is a markdown-first presentation tool for macOS/iOS. The native format is a `.iapresenter` package — a directory bundle containing the slide markdown plus image assets. Slides are authored as plain markdown, where headings appear on slides, paragraphs become speaker notes, and `---` separates slides.

This skill captures the practical workflow: how to bootstrap a deck, how to iterate quickly, and the small handful of layout directives that handle most real-world slide problems.

## Core mental model

iA Presenter inverts the usual presentation tool: **what you say ≠ what you show**. By default, paragraphs are *speaker notes* (only the presenter sees them) and only headings + tab-indented text + images appear on slides. The philosophy: write the talk first as prose, *then* surface select bits onto slides.

Concretely:
- `# Heading` → visible on slide as the title
- `## Subheading` → visible
- `	tab-indented paragraph` → visible body text
- Plain paragraph (no tab) → speaker notes only
- `![](/assets/x.png)` or `/assets/x.png` → image on slide
- `---` (or two blank lines) → new slide

A presenter who spits out bullet-point-heavy slides is fighting the tool. Lean into headlines + one strong visual + speaker notes.

## No official automation surface — edit the bundle directly

iA Presenter has no CLI, no public API, no AppleScript dictionary, no URL scheme, and no Shortcuts integration. Don't go looking — they don't exist. iA's support docs cover only the GUI.

The de facto agent-friendly path is the bundle's plain-text source: `<deck>.iapresenter/text.md`. iA watches the file and hot-reloads on external edits. This skill leans on that.

## The `.iapresenter` bundle

A `.iapresenter` is a directory (macOS treats it as a single file via the bundle bit). Inside:

```
deck.iapresenter/
├── text.md         # the slide markdown — source of truth
├── info.json       # metadata (creator, version, file ID)
├── assets/         # images referenced by text.md
│   └── *.png
└── thumb.png       # auto-generated cover thumbnail
```

**This matters because**: you can edit `text.md` directly with any editor (Vim, an LLM, a script) and iA Presenter hot-reloads. That's the fast iteration path — much faster than clicking around the app.

`info.json` minimum:

```json
{
  "creatorIdentifier": "net.ia.presenter",
  "net.ia.presenter": { "localFileIdentifier": "<UUID>" },
  "transient": false,
  "type": "net.daringfireball.markdown",
  "version": 2
}
```

You can generate one from scratch — `uuidgen` for the ID, drop in `text.md` and `assets/`, and iA Presenter will open it. But the easier path is to **import a `.md` once** (see below) and then edit `text.md` inside the resulting bundle.

## Workflows

### Bootstrapping from a markdown draft (most common)

1. Write `slides.md` in the repo using iA's syntax (headings + paragraphs + `![](image.png)`).
2. In iA Presenter: **File → Import → Markdown / Text**, pick `slides.md`. Don't use File → Open — that filters out `.md` and only accepts `.iapresenter` bundles. (This trips everyone up the first time.)
3. In the import dialog, leave all three checkboxes on: "Convert Markdown to Slides," "Convert Paragraphs into Speaker Notes," "Import local images into .iapresenter file."
4. iA may pop a "Give Access" dialog asking to read the folder containing the images — approve it.
5. Save (Cmd+S) and pick a path inside the repo, e.g. `deck.iapresenter`. Now you have a bundle.
6. From now on, edit `deck.iapresenter/text.md` directly — iA reloads on save.

### Bootstrapping from scratch

If the repo has no `.md` yet, just create `deck.iapresenter/` with the structure above (write `info.json`, drop a `text.md`, make an empty `assets/`). Open it in iA. Faster than the import dance for a known-format deck.

### Iterating on an existing deck

The bundle's `text.md` is the source of truth. Edit it directly with whatever tool — iA picks up changes automatically. To add an image, drop the file into `<bundle>/assets/` and reference it as `/assets/filename.png` in `text.md`.

## Slide markdown — the gotchas

### Import inserts spurious slide breaks

iA's importer **inserts `---` between a heading and a following image at the top of a slide**. So a clean source like:

```markdown
# My slide

![](/assets/chart.png)

speaker notes
```

becomes, after import:

```markdown
# My slide

---

/assets/chart.png

speaker notes
```

…which renders as *two* slides (heading-only, then image-only) instead of one. Fix: open `text.md` in the bundle and delete the spurious `---` lines. Always sanity-check after a first import.

### Image directives go on the line(s) *immediately following* the image path

```markdown
/assets/diagram.png
size: contain
x: center
y: top
```

No blank line between the path and the directives, or they get treated as a separate paragraph (and become speaker notes).

### Useful image directives

- `size: contain` — fit the image without cropping. Use this for diagrams; `cover` (the default) crops aggressively for very wide or very tall images.
- `size: cover` — fill the cell, crop as needed. Default. Good for photos.
- `x: left|center|right`, `y: top|center|bottom` — alignment within the cell.
- `background: true` — image fills the slide as a background; text floats on top. Good for hero shots and full-bleed photos. Bad for diagrams whose detail conflicts with overlaid text. Reach for this when the image is *atmospheric*; avoid when the image is *informational*.

### Slide title patterns

For a title slide with subtitle:

```markdown
# Talk Title
	Speaker name or one-line tagline
```

The tab-indented line becomes a subtitle on the slide.

### Image-then-heading vs heading-then-image

iA's auto-layout picks layout based on element order. Putting the image *first* tends to make it dominant; putting the heading first tends toward heading-left/image-right. Try both if a slide feels off.

## Layout limits — when split layout doesn't fit a wide diagram

Default layout for `# Heading + image` is a horizontal split: title on the left half, image on the right half. For images with extreme aspect ratios (wide system diagrams, tall flowcharts), this halves the available space and the diagram becomes unreadable.

Mitigations, in order of preference:

1. **Re-render the image at a more square aspect ratio.** This is almost always the right answer — tweak the source (mermaid layout direction, matplotlib figsize, etc.) before reaching for layout hacks.
2. **Drop the heading on that slide.** Let the diagram speak for itself; speaker introduces it verbally. Keeps the rest of the deck consistent without fighting the layout engine.
3. **`background: true`** — only if the diagram is OK with a heading overlaid on it. Usually it isn't.

Don't try to override layouts with CSS classes or position pixel-pushing per slide; you'll fight the responsive engine and the deck will break on different displays.

## Themes

Themes are picked via Format → Templates (or the right-hand inspector). iA ships with a solid set: San Francisco, Milano, Copenhagen, Garamond, etc. Each theme handles light/dark and varies fonts + accent colors. Pick one *after* the content is settled — switching is free, so don't agonize early.

For a clean technical/retro deck, the default San Francisco-style theme is usually right. For something more editorial, Garamond. For something dramatic, Copenhagen.

Custom themes are possible (`theme.css` + `template.json`) but rarely worth it — you're better off using a built-in and accepting its constraints.

## Reference: the built-in tutorial

iA ships a tutorial deck at:

```
/Applications/iA Presenter.app/Contents/Resources/Default.md
```

When in doubt about syntax — image positioning, table charts, how to do a TOC slide, etc. — read that file. It's the canonical example of every feature in the tool.

## Anti-patterns

- **Bullet lists on slides.** iA's docs are emphatic: bullets read robotic, increase cognitive load, and tempt the audience to read instead of listen. Convert to a single headline + image, or move to speaker notes.
- **Reading the slide aloud.** If the slide has the line you're saying, the audience reads ahead of you (~250 wpm reading vs ~150 wpm speaking) and tunes you out.
- **Stock imagery.** Per iA: "Stock imagery is worthless and will likely insult your audience's intelligence." Use real screenshots, charts, or nothing.
- **Editing slides only inside the iA UI.** Slow. The text.md hot-reload is the killer feature; use an external editor when you're iterating on prose.

## Quick reference

```markdown
# Title slide
	subtitle (tab-indented)

speaker notes go here as plain paragraphs

---

# Slide with one image

/assets/chart.png
size: contain

speaker notes for this slide

---

# Slide with body text on slide
	A line of body text shown to the audience.
	Another line.

speaker notes
```
