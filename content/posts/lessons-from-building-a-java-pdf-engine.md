---
title: "Lessons from Building a Java PDF Engine"
date: 2026-06-14T20:40:00+01:00
draft: false
slug: "lessons-from-building-a-java-pdf-engine"
description: "Engineering lessons from building layout, pagination, and rendering code in Java."
cover:
  image: "images/posts/lessons-from-building-a-java-pdf-engine.png"
  alt: "A developer looking at layered document layout plans"
  caption: "A document engine is a stack of layout decisions, not only a renderer."
  relative: false
tags:
  - java
  - graphcompose
  - pdf
  - architecture
categories:
  - Engineering
  - GraphCompose
series:
  - Building GraphCompose
---

Building a PDF engine in Java is a useful way to meet a lot of hidden complexity at once. Text measurement, pagination, styling, rendering order, reusable components, API design, testing, and performance all become visible very quickly.

This is not a tutorial about drawing text into a PDF. It is a reflection on what I learned while building the layout engine behind GraphCompose: the hard part is rarely one isolated algorithm. The hard part is making many small rules work together until the final document feels reliable.

## Layout is product behavior

It is tempting to treat layout as a visual detail. In document generation, layout is part of the product contract.

If a section moves to the wrong page, a heading separates from its content, or a table breaks in a confusing way, the output may still be a valid PDF, but it is not a good document.

That changed how I think about layout code. It needs explicit rules, good defaults, and tests around the cases users will actually hit. The engine is not only responsible for drawing. It is responsible for producing a document that a person can trust.

## Layered layout is where assumptions fail

A template usually looks simple with perfect demo data. Real data is less polite.

Names are longer than expected. Tables have one row or one hundred rows. Optional sections appear together. A paragraph that usually fits suddenly needs another page.

Plain pagination is not the hardest version of the problem. If a document is basically flat, with one layer of blocks flowing from top to bottom, pagination is still work, but the model is understandable: measure, split, move overflow to the next page.

The harder case starts when the document has layers and dependencies. One layer may depend on another. A container may need to stretch because of its content. Another block may need to align with it. A visual background, border, sidebar, or overlay may need to resize together with the content. At that point, layout is not only "does this fit on the page?" It becomes "how do all these pieces change together and still land in the right pixels?"

That kind of layered layout forces the engine to answer questions that are easy to ignore:

- Can this block split?
- Should this heading stay with the next block?
- What happens when a section is larger than a page?
- How much spacing is preserved across a page break?
- Which layer owns the final size?
- What depends on what?
- When one element stretches, which other elements must be recomputed?

Those decisions need to be designed, not patched after the fact. Otherwise every new template becomes another special case.

## Some problems are thinking problems

I remember one specific moment when the engine already worked with containers. Blocks could flow, containers could relate to each other, and the basic layout model felt alive. The problem was not that pagination was impossible. If the document had been flat, I could have implemented page flow in a more direct way.

The difficult part appeared when positioning had to work across layered structures. One layer could depend on another layer. A container might stretch because of its children, while another visual element had to follow that new size. Several pieces had to resize, move, and still match each other precisely.

That is where I got stuck.

The code was not completely broken. The idea was not completely wrong. But I could not clearly see how the positioning model should work when size, position, layer order, and dependency rules all had to converge. I added logs, read the logs, changed the logs, and tried to analyze the flow. I also used AI tools to think through the problem, but they could not replace the missing mental model.

For a week, maybe a little more, I almost did not write code. I would sit at the computer, try to write something, and realize I did not know what the next correct line should be. Then I would step away, lie on the sofa, think about the problem, come back, stare at the code again, and repeat the same loop.

That is a strange part of building an engine: sometimes the work is happening, but it does not look like work. No commits. No new classes. No visible progress. Just the mental model slowly turning until it finally clicks.

At some point the shape of the solution became clear. I could see how the positioning should work, where the responsibility belonged, and how to simplify the problem instead of fighting every edge case separately.

Then the work changed completely. Once the idea was clear, the code started to move. Line by line, the implementation began to take shape. The first run did not work everywhere, but it worked in the way I wanted. That first partial success mattered because it proved the pattern.

After that, the task was no longer "what should this be?" It became "how do I shape this code until the pattern is reliable?"

## APIs should express intent

After spending enough time inside the engine, the API problem became clearer too. A user should not need to think in the same low-level mechanics that the engine uses internally.

Good APIs do not hide every detail. They hide accidental complexity and expose meaningful choices.

For a document engine, that means template authors should not need to know every rendering primitive. But they should be able to express layout intent clearly: spacing, grouping, alignment, theme, and behavior around page breaks.

When an API preserves intent, the generated document becomes easier to reason about. The code starts to say what the document is, not only how the PDF backend should draw it.

## Tests need more than one shape

Unit tests are useful, but they are not enough on their own. A layout engine can pass small calculations and still produce a document that looks wrong.

For a PDF engine, I want several kinds of confidence:

- Small tests for layout calculations
- Regression tests for document structure
- Rendered output checks for visual behavior
- Real templates that act as integration examples

The goal is not to test pixels for their own sake. The goal is to catch the kind of changes that would make a business document look broken or untrustworthy.

## Performance work should stay boring

When I was writing the engine, performance was not the first goal. The first goal was simpler and more urgent: make it work. Make it render the document. Make the layout rules actually produce the result they were supposed to produce.

I wrote some parts as quickly as I could because I was trying to keep the whole model in my head. Sometimes I could see that a class or a method repeated something from an earlier version. Sometimes I knew that a calculation would probably be done more than once. But stopping too early to make everything elegant would have broken the thread. In that phase, momentum mattered.

That does not mean performance was ignored. It means the work had phases. First, get the engine to do the job. Then come back to the places where the cost is visible and the behavior is already understood.

By the end of that phase, I had a much clearer idea of where the waste was. I knew which parts had repeated calculations, where layout data could be reused, and which areas would need optimization later.

Most useful performance work then starts with simple questions:

- Are we measuring the same text repeatedly?
- Are we allocating too much inside hot paths?
- Can layout data be reused safely?
- Is the algorithm doing work proportional to the document size?

In Java, boring performance work is often the right performance work: measure first, remove obvious repetition, cache only when the rules are clear, and keep the data structures understandable.

## The main lesson

A PDF engine looks like a rendering problem from the outside. From the inside, it is an API design problem, a layout problem, a testing problem, a performance problem, and a product reliability problem.

That is why it is interesting. The output is only one PDF file, but behind that file there is a chain of decisions about structure, intent, constraints, and trust.
