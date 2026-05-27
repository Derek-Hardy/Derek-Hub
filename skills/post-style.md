---
name: derek-post-style
description: Use this skill when drafting, editing, or extending posts for Derek's Hugo blog so the writing matches the style of existing posts.
---

# Derek's Blog Post Style

## Overall Voice

- Write in first person when explaining motivation, preference, or experience.
- Sound practical, curious, and reflective rather than formal or polished-marketing.
- Prefer "I want to put down a summary", "From my experience", "I find", "I think", and similar direct personal framing.
- Make technical judgment visible: explain what is useful in production, what is rarely used, what feels error-prone, or what is worth remembering.
- Keep the tone friendly and occasionally conversational, with light asides when natural.

## Common Post Shape

Most technical posts follow this flow:

1. Start with 2-4 short paragraphs explaining why the topic matters.
2. Use `###` sections for each concept, feature, function, pattern, or tool.
3. Explain the concept in plain language before showing code.
4. Add concrete examples with comments.
5. Compare with another language, old approach, or production concern when it clarifies the point.
6. End with a practical takeaway, future note, or brief personal conclusion when useful.

For longer walkthroughs, `##` sections are acceptable, as in the HTTP post. For most list-style technical notes, prefer `###`.

## Technical Writing Patterns

- Anchor explanations in real usage, not abstract definitions alone.
- Introduce code with phrases like:
  - "Now, we want to..."
  - "For example..."
  - "It looks self-explanatory..."
  - "Now let's use it in action..."
- Use Markdown tables for compact comparisons of types, functions, decorators, or behavior.
- Use blockquotes sparingly for official docs or memorable source quotes.
- Link to related internal posts with relative links when there is a natural connection.
- Mention version numbers when discussing language features, especially Go and Python releases.

## Code Style In Posts

- Include many fenced code blocks; code examples are central to the style.
- Use language labels such as `go`, `python`, `python3`, `java`, `shell`, `bash`, and `text`.
- Prefer small runnable or near-runnable snippets over pseudocode.
- Add inline comments to explain the important line, especially where syntax may be surprising.
- Show the older or clumsier approach first when the post is about a newer improvement.
- For larger code examples, use the existing Hugo shortcode:

````markdown
{{<details title="To see the full source code" >}}
```go
// code here
```
{{</details>}}
````

## Topic Tendencies

- Core engineering topics: Go, Python, HTTP, data structures, concurrency, design patterns, Git productivity.
- Personal interest posts: language learning, YouTube channels, practical resources, travel/culture experience.
- Derek often compares Go with Java or Python, and Python with Go, to explain why a feature feels intuitive or awkward.
- Production-minded concerns appear often: observability, maintainability, latency, error handling, resource usage, clean commit history.

## Language Learning Post Pattern

- Begin with a short personal story about why the language mattered.
- Explain the practical learning goal: daily vocabulary, accents, conversations, culture, or real-life exposure.
- Use `### [Resource Name](url)` headings for each channel/resource.
- Give 2-4 short paragraphs per resource describing what it is good for and why Derek likes it.
- Keep the tone warmer and more casual than technical posts.

## Style To Preserve

- Short paragraphs with clear line breaks.
- Direct vocabulary over academic phrasing.
- Explanations that move from intuition to example.
- A learner's notebook feel: useful reminders, personal observations, and examples Derek can reuse later.
- Mild imperfections and natural phrasing are acceptable; do not over-sanitize into corporate prose.
