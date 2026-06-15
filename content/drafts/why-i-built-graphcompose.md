---
title: "Why I Built GraphCompose"
date: 2026-06-14
draft: true
slug: "why-i-built-graphcompose"
description: "The motivation behind a declarative Java and Kotlin document layout engine."
tags:
  - java
  - kotlin
  - graphcompose
  - pdf
  - open-source
categories:
  - GraphCompose
  - Open Source
---

Most backend systems eventually need to produce documents: invoices, reports, offers, letters, CVs, certificates, export packs. The business expects those documents to be reliable, readable, branded, and easy to change.

The code often tells a different story.

PDF generation in Java can become a long sequence of coordinates, drawing calls, conditionals, and layout patches. A small content change can break spacing. A new section can disturb pagination. A template can look fine in one data case and fail in another.

I built GraphCompose because I wanted document generation to feel more like building a product UI and less like manually painting pixels.

## The problem I wanted to solve

The real problem was not "how do I draw text into a PDF?" Java libraries can already do that.

The harder problem was:

- How do I describe document structure clearly?
- How do I reuse sections across templates?
- How do I keep layout decisions visible in code?
- How do I support Java and Kotlin developers without forcing a web stack into document generation?
- How do I make documents testable enough to trust?

I wanted a document engine where the template describes intent first: sections, blocks, spacing, typography, hierarchy, and reusable components. Rendering should still be precise, but precision should not require every template author to think in raw coordinates.

## Why declarative layout

Backend developers already understand declarative ideas from SQL, Spring configuration, validation rules, and build files. You describe what should exist, and the engine handles the mechanics.

GraphCompose applies that style to documents.

Instead of making the template responsible for every drawing step, the template should define the document model and layout rules. The engine should handle measurement, pagination, rendering, and repeatable behavior.

That separation matters because document templates change constantly. A declarative template is easier to review, easier to refactor, and easier to explain to another developer.

## Why this matters for backend work

Documents are often part of serious business flows. They are not decoration. A generated PDF might be the thing sent to a customer, attached to an application, used by an operations team, or archived for compliance.

That means the document engine is backend infrastructure.

It needs clear APIs, predictable output, versionable templates, and tests. It needs to be boring in the best way: understandable, repeatable, and safe to evolve.

GraphCompose is my attempt to build that kind of tool for Java and Kotlin developers.

## What I am still learning

Building a document engine is a good reminder that "simple API" does not mean "simple implementation." Layout, pagination, typography, and rendering all contain edge cases.

The work is worth it because every hard decision in the engine can remove complexity from application code.

That is the trade-off I wanted: keep the hard parts in one focused library, so product code can describe documents clearly.
