---
title: "GraphCompose 2.1: One Document, Two Real Outputs"
date: 2026-07-26T10:00:00+01:00
draft: false
slug: "graphcompose-2-1-twinned"
description: "The twinned release: the same resolved layout now renders as a print-ready PDF and as an editable PowerPoint deck — native shapes, not a screenshot of a page — plus keep-with-next pagination and a render that can no longer destroy the file it replaces."
cover:
  image: "images/posts/graphcompose-2-1-twinned.png"
  alt: "One document description resolving into a layout graph, then into a PDF page and an editable slide deck"
  caption: "One description, one layout calculation, two backends that consume the same resolved geometry."
  relative: false
tags:
  - java
  - graphcompose
  - pdf
  - pptx
  - architecture
  - release
categories:
  - GraphCompose
  - Engineering
series:
  - Building GraphCompose
---

My girlfriend is a senior marketing researcher, and last year she started building presentations with AI tools. The first result was usually fine. The layout was there, the content was placed, the slides looked finished.

Then she needed to fix a typo.

Not restructure the deck — fix one word. And because the tool owned the whole artifact, the only way to change one word was to ask for the whole deck again. It would fix the typo and move a box. Or rewrite a sentence that was already correct. Or quietly break a slide that had been fine. A thirty-second correction became a negotiation with a generator.

The file looked like a presentation. It was not one. In most of these pipelines the slide is composed through HTML or SVG and then flattened — an image of a deck, wearing a `.pptx` extension. It is fast, and it does not belong to the person holding it.

I was building a PDF engine at the time, for what looked like an unrelated reason, and it turned out to be the same problem.

## Coordinates belong to the layout system

GraphCompose started because I wanted to write my CV in Java and ended up hand-calculating offsets in PDFBox until I asked why I was doing arithmetic a program is better at. I told that story in [I Just Wanted to Make a CV in Java]({{< relref "/posts/i-just-wanted-to-make-a-cv-in-java.md" >}}), so the short version: nobody thinks about a document in X and Y. They think in sections, headings, tables, panels, and what sits above what. Coordinates are an implementation detail, and they belong to the engine.

The consequence is a stack of separate layers. Data is one. Document structure is another. Theme, colour, and the visual patterns a company reuses are another. The output format is the last one, and it is the thinnest.

Which gives you the property that matters: changing content should not rebuild the design, changing the theme should not touch the data, and adding an output format should not mean writing a second layout engine. Above all, correcting one word should not regenerate the document.

[2.0](https://github.com/DemchaAV/GraphCompose/releases/tag/v2.0.0) turned that into packaging — a lean core that owns the document model, measurement, pagination and resolved geometry, with render backends as separate modules discovered at runtime. PDFBox draws PDF. Nothing in the core knows that.

2.1 is where the second backend arrives and the architecture has to prove it was worth the trouble. I called the line *twinned*.

## The deck is not a picture of the page

`graph-compose-render-pptx` does not render a PDF and paste it onto a slide. It consumes the *same resolved layout graph* the PDF backend consumes. One page becomes one identically-sized slide, and every element keeps its position by construction rather than by a second implementation that has to agree.

```java
Path deck = Path.of("twin-output.pptx");
try (DocumentSession doc = GraphCompose.document(Path.of("twin-output.pdf"))
        .pageSize(DocumentPageSize.SLIDE_16_9)
        .create()) {
    compose(doc);        // one description
    doc.buildPdf();      // print-ready PDF
    doc.buildPptx(deck); // editable PowerPoint
}
```

Paragraphs land as text frames seated on the measured baselines. Tables land as row fills, border edges and per-cell frames — across page breaks, with repeated headers and row spans. Shapes, polygons, free paths with gradient fills, images, barcodes and transform groups are native DrawingML. Headers, footers, watermarks, hyperlinks, internal slide jumps and bookmark names come along. The reference page in the README lands as **69 native shapes**, and the only picture on it is a clip-masked logo.

So you can open it, click the headline, and type. Move a box before a meeting. Fix the typo. Nothing regenerates.

There is a DOCX exporter too, but it is a different animal: a semantic export that walks the node tree with no layout pass at all. PDF and PPTX are the pair that share resolved geometry, and that sharing is the whole claim.

## Where it says what it cannot do

A backend that shares geometry does not share every capability, and I would rather publish the gap than let someone discover it in front of a client.

There is a [capability matrix](https://github.com/DemchaAV/GraphCompose/blob/main/docs/architecture/backend-capability-matrix.md) tracking 38 things a document can contain. For PPTX, 24 map to a native equivalent and 4 are unsupported outright. The remaining 10 are partial, and not in the same way — some render natively with an approximated detail (distinct per-corner radii collapse to one, a numeric dash array snaps to the nearest preset, a radial gradient takes the closest DrawingML shade), while others lose something the format genuinely cannot hold.

Clipping is the honest one. DrawingML has no graphics-state clip, so a clip region that actually cuts ink is sub-rendered through the PDF backend and placed as one transparent picture on the clip bounds. Pixel-exact, not editable as shapes, and run-level link hotspots inside those bounds are not emitted. A clip that provably cannot remove ink — a rounded card whose padded content never reaches the corners — skips the fallback and stays native. The backend logs how many regions it rasterized and how many megapixels they cost, because each one is a full second render and a hole in the editability you came for.

Charts are not native PowerPoint chart models. Since 1.8 a chart [compiles down to engine primitives]({{< relref "/posts/graphcompose-1-8-illustrative.md" >}}) at layout time rather than being a special kind of object, which turned out to be a gift here: the chart arrives as editable vector geometry, drawn from the source data, without a single line of chart-specific code in either backend.

Fonts are the last one, and they are not really the engine's decision. Both backends measure through the same pipeline, but PowerPoint draws the glyphs. Families are embedded where the licensing bits allow it, and when a family has to be substituted the backend warns once per family at build time — so a deck that will look different on someone else's machine tells you before you send it.

## The bug that proves the pipeline is shared

My favourite item in the release is a fix.

A span naming a standard-14 style variant — `Helvetica-Bold`, `Times-Italic` and their siblings — was travelling to PowerPoint as a bold or italic run flag. Those names are family aliases: the engine resolves each one to its regular base and takes the real face from the span's decoration. So layout had measured regular metrics, and PowerPoint then drew a face about 6% wider than the slot reserved for it. A chip label pushed past its card. Two words of a rich-text heading closed the gap between them until they read as one.

Six per cent. It is the kind of bug you only get to have when two backends are supposed to agree — with two independent layout implementations, nobody would have noticed a divergence, because there would have been nothing to diverge from. Sharing a measurement pipeline means a mismatch is a defect rather than a fact of life.

The same pass fixed the mirror image: the PDF backend was resolving `UNDERLINE` and `STRIKETHROUGH` to font faces that alias back to the regular program, so decorated text rendered as plain glyphs — while PPTX had been drawing real marks all along. PDF now draws em-proportional bands in the run's colour, on paragraph runs, chips and table cells alike.

## A failed render should not destroy what it replaces

Not everything in 2.1 is about slides.

`buildPdf(Path)` and `buildPptx(Path)` used to open the destination before the backend produced a byte — which means they emptied it. An oversized node, a missing backend, a full disk, and you were left with a zero-byte file where a published document had been. The write is now staged into a scratch file in the destination's own directory and moved onto the destination only after the render returns: atomic where the filesystem supports it, so a concurrent reader never sees a half-written document, and the existing file keeps its permissions.

This is the least exciting paragraph in the release and possibly the most important one. A document generator is [backend infrastructure]({{< relref "/posts/documents-are-backend-infrastructure.md" >}}) — it runs unattended, on a schedule, over data nobody looked at first — and infrastructure is judged by what it does when it fails, not when it succeeds.

In the same spirit: two backends registered for the same format used to let classpath order pick the renderer. It now throws and names both competing providers.

## Headings that stay with their content

1.8 added `keepTogether()` — move this whole block to the next page rather than split it. The obvious sibling was missing, and 2.1 adds it: `keepWithNext()`, which says *do not strand this heading at the bottom of a page without the thing it introduces*.

The subtlety is what counts as "the thing it introduces". If the following block is a page-spanning table, requiring the whole table to fit would be useless. So the rule is the heading plus the first slice — a paragraph's first line, a table's repeated headers plus its first body row, a list's first item — which makes it behave sensibly whether what follows is atomic or paginates. It is off by default, inert when nothing follows, and best-effort when even the heading plus one line cannot share a page.

Every single-column CV preset and the Modern Proposal template now use it. Section titles stop appearing alone at the bottom of a page, which is the sort of thing nobody praises and everybody notices.

## The principle underneath

A business presentation is usually not a permanent product. It says what happened, why, and what the company should do next. It needs to be clear, consistent with the patterns the company already uses, and fast for the room to read. Making it should support the work, not become the work.

A researcher who spends an afternoon arguing with a generator over one sentence has been converted from an analyst into an operator of a tool. That is a bad trade, and it is an architectural failure, not a UX one — it happens because the tool owns the artifact and the person only owns the prompt.

The PPTX backend is `@Beta`; the API shape may still move, though the geometry relationship with the PDF backend is a design invariant and will not. But the release does the thing the architecture was built to do:

Describe the document once. Let the engine resolve the coordinates. Render it through backends that share that resolution. And hand back files that are real, editable, and yours.

[GraphCompose 2.1.0](https://github.com/DemchaAV/GraphCompose/releases/tag/v2.1.0) is out, with the exhaustive list in the [changelog](https://github.com/DemchaAV/GraphCompose/blob/main/CHANGELOG.md).
