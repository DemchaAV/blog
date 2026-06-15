---
title: "What Drove Me to Build GraphCompose"
date: 2026-06-14T21:00:00+01:00
draft: false
slug: "what-drove-me-to-build-graphcompose"
aliases:
  - "/posts/why-i-started-building-graphcompose/"
description: "From fighting PDFBox coordinates to building a Java-first declarative document layout engine."
cover:
  image: "/images/posts/what-drove-me-to-build-graphcompose.png"
  alt: "A developer connecting layout fragments into finished document pages"
  caption: "GraphCompose grew from one idea: describe intent, resolve layout, render the document."
  relative: false
tags:
  - java
  - pdf
  - graphcompose
  - architecture
  - open-source
categories:
  - GraphCompose
  - Open Source
series:
  - Building GraphCompose
---

It started with a simple idea: I wanted to create my own CV in pure Java.

No big framework. No designer tool. No attempt to overengineer everything before I even had one page. Just Java, PDFBox, and the confidence every developer has right before a supposedly small task becomes a real project.

At first I thought: how hard can it be?

Very quickly I found myself inside the classic PDF generation loop:

- write coordinates
- render the PDF
- open it
- notice that something is slightly wrong
- change the coordinates
- render again
- repeat until the document looks acceptable

One block was too low. One line was too close to another. One section did not fit. One font change broke the page. The funny part was that nothing was technically wrong. PDFBox was doing exactly what I asked.

The problem was that I was asking in the wrong language.

I was talking to the PDF like a machine:

> "Draw this text at x = 42, y = 713."

But in my head I was thinking like a human:

> "I want the title at the top, centered, with a strong font. Then a divider. Then a section. Then some text. And if the content grows, move things properly."

That gap became the beginning of GraphCompose.

## Coordinates Are Not Intent

When we imagine a document, we do not imagine coordinates. Nobody thinks, "I want my professional summary at x 58 and y 492." We think in hierarchy, structure, spacing, sections, emphasis, and meaning.

That was the first real insight: low-level PDF libraries think in drawing commands, but humans think in layout intent.

The document I wanted was not a list of coordinates. It was a structured object:

- a header
- a section
- a paragraph
- a table
- a block that may move to the next page
- a visual system that should survive changes in data

So GraphCompose did not start as "make PDFBox easier." That would be too small. The deeper idea was this:

> What if Java developers could describe a document the way they actually see it?

Not coordinates first. Structure first. Intent first. Then the engine can calculate the boring and fragile part.

## From PDF to Layout

At the beginning, GraphCompose was close to the PDF world. That made sense because the first target was PDF output. But the more I worked on it, the more obvious it became that coordinates are not only a PDF problem.

PDF has coordinates. Images have coordinates. SVG has coordinates. Slides have coordinates. Even UI rendering becomes coordinates at some point.

So the better question was not:

> "How do I draw this with PDFBox?"

It was:

> "How do I describe the document once, resolve its layout, and let a backend draw it?"

That changed the direction of the project. The engine should understand documents. The backend should understand drawing. PDFBox can draw PDF. Apache POI can write DOCX. A future backend could create slides, SVG, or images.

Different output. Same resolved layout idea.

That separation is what made GraphCompose more interesting than "my CV generator."

## The ECS Phase

Before the semantic DSL became the main public surface, I experimented with an ECS-style architecture. I still think that was a strong step for the engine internals.

ECS gives you a flexible way to model objects. You have entities as identities, and then you attach components: size, padding, margin, content, placement, render markers, parent relations, and children order. Systems process those components. One system measures, one lays out, one paginates, and one renders.

That structure is powerful because each part has a job. If you want a new property, you add a component. If you want new behavior, you teach a system how to work with that component. If you want a new primitive, you model it as an entity with the right set of components.

For example, if the engine already knows how to handle an atomic rectangle, adding a circle is not a completely new universe. The shape payload is different, but from the layout engine's point of view both objects take space, can move to another page, and can be rendered by a handler.

ECS gave the project flexibility.

But it also exposed a problem: ECS is a good engine brain, not always a good public API.

Part of the complexity was hidden from me because I already had the engine model in my head. I knew how containers worked, how components could be layered, how entities could be connected, and how the systems would process the tree. For me, that mental model was available before I wrote the API.

For another developer, that is not true. If someone has not been living inside the engine, they first need to understand the entity/component model, the container structure, the parent-child relationships, and the rendering pipeline before they can feel productive. That is a lot of preparation before writing one useful document.

There was also a lifecycle question. In the ECS version, the entity tree still had to be assembled for each generation run. The benchmarks showed that this was fast, so it was not a practical bottleneck at that stage. But it still meant repeated construction of the same kind of object graph.

A more semantic approach can make more of the template structure stable: describe the document shape once, bind data for a specific render, resolve layout, and then draw the output. That does not automatically make everything faster, but it gives the engine a better path toward reuse, less repeated setup, and clearer optimization boundaries.

Most developers do not want to build a CV by manually creating entities and attaching component sets. They want something closer to this:

```java
document.pageFlow(page -> page
    .module("Professional Summary", module -> module
        .paragraph("Backend developer with Java and Spring Boot experience.")));
```

That difference matters. Engine internals can be technical. The public API should feel calm.

## The Semantic DSL Shift

The next step was moving GraphCompose toward a semantic document model.

Not because ECS was wrong. ECS did its job. But I needed a higher-level layer because writing documents should feel like writing documents.

A CV should have modules. A proposal should have sections. An invoice should have tables. A report should have paragraphs, images, dividers, and page breaks. Application code should describe semantic nodes, and the engine should compile them into layout fragments.

That became the shape of the architecture:

- application code describes document intent
- the engine resolves layout and pagination
- the backend renders the final output

Before that shift, the engine was closer to "objects on a page." After the shift, it became closer to "document meaning that becomes objects on a page."

That opened the door to stronger testing, reusable templates, theme-driven design, custom node definitions, multiple output backends, and safer internal refactoring.

The public side became simpler. The inside became more structured. That is usually a good sign.

## Templates Made It Feel Like a Product

Templates were another important step.

At first, templates can look like examples. But they are more than that. Templates are a stress test for the architecture. If every template turns into a long file full of duplicated header logic, spacing hacks, hardcoded colors, and one-off rendering decisions, then the engine is not helping enough.

That is exactly what I started to see in GraphCompose. In the beginning, I treated templates mostly as written code: build the template, show how the engine works, and let people reuse parts if they wanted to. That was useful for proving the idea, but it was not enough for a real user experience.

The classes became large. It was easy to imagine writing templates forever in the same style: repeat a header here, repeat a block there, copy some spacing logic, duplicate visual decisions, and slowly create a set of templates that all looked different on the surface but shared the same hidden mess underneath.

For a user who comes to the library with a specific task, that is not what they need. They do not want to read all the internal template code before they can create a CV or a business document. They usually need data, visual options, small configuration points, and maybe a controlled way to adjust existing components.

They should not have to touch the internal template implementation just to get a useful result.

That changed how I started thinking about templates. A template is not just an example. A template is closer to a product surface. If someone opens the library, chooses a CV template, and wants to create their own document, the ideal path should be simple: fill the data, choose the available options, and generate the output.

The user should not be forced into the code path where they can accidentally break layout internals while trying to change content.

The newer direction is cleaner:

- theme tokens
- layout slots
- reusable components
- structured spec data

That matters because a CV is not just "draw some text." A CV becomes data + layout + theme + components. A cover letter can share the same visual language. An invoice or proposal can reuse the same design tokens.

That is when GraphCompose started to look less like a rendering experiment and more like a document generation platform.

It is still young. It is still not perfect. Some parts are still being cleaned. But the direction is much clearer.

## Open Source Changed the View

One of the useful moments came from the outside.

A user found the library, got interested, and asked a simple question: why is there no Java 17 support?

He was right.

I had been focused on layout, rendering, pagination, performance, tests, templates, and internal architecture. Those are important things. But from a user's point of view, Java 17 support also matters because many real production Java stacks are still on Java 17.

That is where open source became valuable. A fresh view revealed an adoption problem I had not prioritized enough. Then @jottinger helped move the project toward Java 17 compatibility, refreshed dependencies, and improved the project's practical usability.

That kind of contribution is not noise. It is exactly why open source is useful. Someone looking from the user side can see something obvious that the builder misses from inside the engine.

## The Real Idea

For me, the real idea of GraphCompose is simple:

Documents should be described like documents, not like manual drawing scripts.

A developer should be able to say:

- I want a section
- I want a paragraph
- I want this table to repeat headers
- I want this block to move to the next page if needed
- I want this shape to contain content
- I want this output as PDF now, and maybe another backend later

Coordinates are necessary, but they should be the engine's problem. They should not be the developer's problem every time a font size, paragraph length, or section order changes.

That is why I kept building GraphCompose.

It started because I wanted to make a CV. Then PDFBox made me fight coordinates. Then that frustration became architecture.

Now the direction is clearer:

> Describe intent. Resolve layout. Render anywhere.

That is the part I find exciting.
