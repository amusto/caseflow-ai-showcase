# Graphic Spec — Article 01: AI-Ready SDLC Overview

## Use case

Hero image for the LinkedIn post and the long-form article. Doubles as the link-preview thumbnail when the article URL is shared.

## Dimensions

- **Primary**: 1200 × 627 px (LinkedIn link-preview ratio, 1.91:1)
- **Optional square**: 1080 × 1080 px for standalone LinkedIn posts where a square crops better in the feed.

The SVG at `graphic.svg` is authored at the 1200 × 627 dimensions and uses a `viewBox` that lets it scale cleanly.

## Concept

A horizontal flow visualizing the **AI-Ready SDLC loop** — the cycle that the article argues every AI-native team needs.

Six stages, left-to-right, with a feedback arrow looping the last back to the first:

```
Decision  →  Spec  →  Generate  →  Verify  →  Ship  →  Audit
                                                          │
                                                          └──── loops back to Decision
```

Each stage is a labeled node. The flow direction is unambiguous (single arrow style, no branches). The loop arc at the bottom makes the cyclical nature obvious without crowding the linear flow.

## Visual style (revised 2026-05-20)

- **Palette** — light blue-gray background (`#F5F7FA`), dark navy text (`#1A2332`), accent in CaseFlow's MUI primary blue (`#1976D2`) for the arrows, node borders, and step numbers. White (`#FFFFFF`) node fills for high contrast.
- **Why the palette change** — the previous dark navy version felt heavy in isolation. The light palette pairs better with screenshots of the actual deployed app (which uses the same `#1976D2` brand color), letting the methodology diagram and product screenshots feel like one cohesive visual system.
- **Typography** — sans-serif, modern. SVG uses `Inter` with a generic `system-ui` fallback. Title is uppercase tracking-wide for an editorial feel; node labels are sentence case.
- **Geometry** — rounded rectangles for nodes, single-weight arrows, generous whitespace. No gradients, no skeuomorphism, no drop shadows.
- **Headline** — `AI-READY SDLC` in the top-left. Subhead: `The loop that separates "using AI" from "building with AI."`
- **Attribution** — `armando.musto / caseflow-ai` in muted gray, bottom-right. Keep this subtle.

## Two-image LinkedIn post strategy

For Article 01, the LinkedIn post attaches **two images** (LinkedIn carousel), not one:

1. **Image 1** — `graphic@2x.png` (this SVG exported). The methodology hook.
2. **Image 2** — `dashboard-screenshot.png` from the deployed `caseflow.musto.io`. The receipt that the methodology produces real work.

Readers scroll Image 1 → Image 2 and get the full "here's the thinking + here's the artifact" arc. This pattern should be the default for any article that has both a conceptual diagram AND a deployed surface to point at. Future articles in the series should follow the same dual-image template wherever a real screenshot is available.

## Tone

Editorial / publication-grade. The graphic should feel like it came from the cover of a thoughtful engineering newsletter, not a marketing deck. No clip-art icons; the typography and flow do the work.

## File deliverables

- `graphic.svg` — authored SVG, ready to use directly on LinkedIn or to convert to PNG via any SVG-to-PNG tool.
- (Optional, to add later) `graphic.png` — 1200 × 627 PNG export at 2× resolution for retina displays.

## Conversion / export notes

LinkedIn accepts SVG for some placements but PNG/JPG for in-feed images. To export:

```bash
# Option A: rsvg-convert (Homebrew: `brew install librsvg`)
rsvg-convert -w 2400 -h 1254 graphic.svg > graphic@2x.png

# Option B: Inkscape headless
inkscape graphic.svg --export-type=png --export-width=2400 --export-filename=graphic@2x.png
```

## Iteration notes

The current SVG is a v1 — the intent is to look intentional and minimal. If a designer iterates, the goals to preserve:

1. **Six labeled stages, left-to-right, with a loop.** That's the entire idea. Don't break it.
2. **Editorial restraint.** No emoji icons inside the nodes, no drop shadows, no gradient fills.
3. **Readable at thumbnail size.** Test at 320 × 167 (LinkedIn mobile feed scale). Node labels must still be legible.
