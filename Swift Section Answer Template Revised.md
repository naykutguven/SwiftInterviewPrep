---
tags:
  - interview-prep
  - swift
  - ios
  - "{{TOPIC_TAG}}"
---

# {{SECTION_ID}} — {{SECTION_TITLE}}

**Goal:** {{One observable skill from the rubric's selected scope.}}

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]].

<!--
Authoring guidance — remove this comment and unused placeholders from a finished guide.

- Explain the mechanism, its consequence, and the reader's next decision in plain language.
- Aim for 600–900 words of core prose, excluding code and references. This is a guide, not a quota or hard limit.
- Give each explanation one home. Avoid separate spoken answers, level-specific scripts, repeated summaries, or duplicate flashcard answers.
- Cover the essential rubric questions and focused exercise. Repeat rule/question blocks as needed; link substantial extensions.
- Omit the worked example if the exercise already demonstrates the same behavior. Adapt code probes to configurations, diagnostics, or measurements for tooling topics.
- Make examples standalone or identify shared declarations. State baseline overrides, relevant availability, and whether code runs or is intentionally rejected. Separate guarantees from observed results.
- Omit redundant nonisolated modifiers. Use explicit MainActor isolation only when the scenario requires it.
- Prefer descriptive inline links to official sources beside the claims they support, using a specific section where possible. Avoid repeating the same link in consecutive sentences.
- Keep Official sources as a consolidated, deduplicated list, including sources linked inline. Use [[Guide]] or [[Guide#Section]] for internal references and Markdown links for external sources.
-->

## Mental model

{{Explain what the concept does and why it works. Connect the mechanism to a practical problem, then state the most useful guarantee or limitation. Keep paragraphs short.}} [{{Supporting official section}}]({{SOURCE_URL}})

## Rules and pitfalls

### {{Rule or decision}}

{{State the rule or decision criterion and explain its practical consequence.}}

**Common pitfall — when relevant:** {{Describe a likely mistake, why it fails, and the correct approach. Use a small example only when helpful. Omit this block when it adds no distinct lesson.}}

## Interview questions

### {{Essential rubric question}}

{{Answer directly in the first sentence. Add only the reasoning, qualification, or example needed to understand the answer. Use these questions to apply the mental model rather than repeat its definition.}}

## Worked example

**Setup:** {{Imports, shared declarations, relevant settings, and run/compile-only status.}}

**Predict:** {{One question about behavior, validity, ownership, dispatch, isolation, or cost.}}

```swift
{{Small, focused example.}}
```

**Outcome:** {{Output, warning, compiler rejection, trap, or permitted outcomes. Give exact output only when guaranteed; describe a diagnostic's meaning rather than depend on its wording.}}

**Why:** {{Trace the operations that explain the outcome. Add a diagram only if it makes the relationship easier to understand.}}

{{If the example is defective, show a minimal correction and explain the rule it restores. Otherwise, omit a fix. Include an alternative only when it exposes a meaningful tradeoff.}}

## Focused exercise

**Task:** {{Use the rubric's focused exercise. State the requirement and relevant constraints without expanding it into a larger application.}}

**Acceptance checks:**

- {{Required observable behavior.}}
- {{A relevant boundary or failure case.}}

**Solution:**

{{Show the implementation, configuration, or reasoned walkthrough appropriate to the task. Include a test or verification step when needed to establish correctness.}}

**Reasoning:** {{Connect the important decisions to the rules above. Explain a material tradeoff or limitation without restating the solution.}}

## In practice

**Choose it when:** {{Give a concrete decision criterion and explain when a simpler alternative would be sufficient.}}

**Investigate by:** {{Name one useful observation, tool, or check that helps diagnose a likely failure.}}

**Deeper follow-up — optional:** {{Ask one question about an API boundary, migration, ownership, isolation, or measured cost.}}

**Strong evidence:** {{Describe the decision and validation that would demonstrate deeper judgment, using the rubric's score-4 anchor. Omit this and the follow-up when unnecessary.}}

## References

### Related guides

- [[{{RELATED_TOPIC_GUIDE}}]] — {{Explain the connection or prerequisite.}}

### Official sources

- [{{Source title and specific section}}]({{SOURCE_URL}}) — {{Identify the rule or claim supported.}}
