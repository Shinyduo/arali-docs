---
name: arali-docs-metadata
description: Write or update only meta titles and meta descriptions for Arali developer documentation pages at docs.arali.ai.
---

# Arali Docs Metadata Skill

Use this skill **only** to write, review, or update a documentation page's **meta title** and **meta description**.

## Rules

1. **Meta title: 50–60 characters**, including spaces and punctuation.
2. **Meta description: 140–160 characters**, including spaces and punctuation.
3. Read the page first. Both fields must accurately describe its actual content.
4. Include the page's primary topic or keyword naturally, preferably near the start of the title and in the description where it fits.
5. Make titles and descriptions clear, specific, useful to developers, and unique across pages. Avoid keyword stuffing, generic filler, exaggerated claims, and invented functionality.
6. Count the characters of **each final field** and revise until both are within their required ranges. Never treat the limits as optional.

## Applying metadata

- For a Mintlify `.md` or `.mdx` page, use the existing `title` and `description` fields in YAML frontmatter. Do not add any other metadata fields.
- For API pages generated from an OpenAPI spec, identify the existing source of the **displayed page metadata** and update only the supported title/description metadata there. Do not change API definitions to satisfy a character limit.
- Mintlify's `title` can also affect visible headings or navigation. If changing it would alter the page UI unexpectedly, **propose the metadata and flag this before editing**.
- Do not edit body copy, headings, URLs, navigation, code examples, schemas, canonical URLs, robots settings, social tags, or any other content/configuration.

## Example

```yaml
---
title: "Arali API Documentation: Authentication and Requests"
description: "Learn how to authenticate requests and understand Arali API responses using the developer documentation, with guidance for common integration tasks."
---
```

Example title: **52 characters**. Example description: **148 characters**. The example is illustrative; write metadata based on each actual page.

## Output

For each page, report: **page path | meta title | title character count | meta description | description character count**.

If asked only to draft or review, do not edit files. If asked to apply, edit only the two metadata fields and report what changed.
