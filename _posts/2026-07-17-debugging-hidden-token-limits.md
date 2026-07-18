---
title: "Hidden limits masquerading first as a database problem and then a serialization problem"
date: 2026-07-17
categories: [projects]
tags: [java, spring-ai, debugging, ai-agents]
excerpt: "A tool call kept arriving with null arguments. The fix had nothing to do with the code that was failing."
---

I'm building an autonomous research agent - Java, Spring Boot, Spring AI, Claude - that searches a private document collection and writes up findings. Partway through, I hit a bug that took me through three wrong hypotheses before I found the real one. I think the path getting there is more useful than the fix itself.

## The symptom

The agent had a tool, `writeAgentReport`, that persists a research report to Postgres. It takes three arguments: the original goal, the full report text, and a list of source document IDs. Straightforward.

Except it kept failing:

```
Exception thrown by tool: writeAgentReport. Message: not-null property 
references a null or transient value: claudeagent.model.AgentReport.reportText
```

The `agentGoal` argument arrived correctly every time. `reportText` and `sourceDocumentIds` arrived as `null` - every single time, with no variation. And the agent kept retrying the exact same call, over and over, roughly every 6–7 seconds, always failing the same way.

## What hadn't happened part one: it's a database column mapping problem

This felt familiar. Earlier in the project, I'd hit a real type-mismatch bug - Postgres rejecting a `byte[]` because Hibernate was sending it as `bytea` when the column was declared `vector`. So my first instinct was: same category of bug, different field. I checked the entity, the converter, the column definitions. All correct. Dead end.

## What hadn't happened part deux: it's a multi-parameter binding problem

Three tool parameters, only the first one arriving correctly - that pattern looked suspicious. I collapsed the tool signature down to a single combined string parameter, just to see if the problem was specific to binding multiple arguments, or to the `List<String>` type specifically.

It still arrived null. That ruled out the parameter-shape theory entirely - if a *single* string parameter came through empty, this had nothing to do with how many parameters there were or what type they were.

## What still hadn't happened part trey: Claude isn't generating the arguments at all

At this point I turned on debug logging for the actual tool-calling sequence. The logs showed something I hadn't expected: this wasn't one failed call being silently retried by some infrastructure layer. Each attempt showed a fresh `ChatResponseMetadata` entry and a fresh call to Claude - meaning the model was being asked again, genuinely, each time, and generating the same broken result every time.

That ruled out "stale retry replaying old arguments." Whatever was happening, it was happening fresh, identically, on every single attempt.

## What had happened was

`max-tokens` had never been explicitly set.  My naive assumption was that without setting a limit, there wouldn't be one - and I was careful to keep my Anthropic API key set to not auto-reload and kept the balance on it at or under $10 just in case things went off the rails.  It turns out that Spring AI doesn't leave that unset to just let Anthropic's API decide. It has its own hardcoded fallback, `AnthropicChatModel.DEFAULT_MAX_TOKENS`, defined directly in the library's source since at least milestone `1.0.0-M4`. I checked to see what mine had actually been running on: **500 tokens.** Roughly 375 words, for the entire turn - reasoning, tool call structure, and argument content combined.

`writeAgentReport` was the fourth tool call in the sequence - after a document search and two chunk lookups. By that point, Claude had already spent some of that 500-token budget on the reasoning and tool-call structure for the earlier steps in the turn. When it got to constructing the final call - the one requiring a long, synthesized report as an argument, explicitly described in the tool's own documentation as "typically several paragraphs" - there was nowhere near enough budget left to finish generating a complete, valid JSON payload before hitting the ceiling.

Truncating a JSON payload mid-string doesn't produce a "partial string." It produces something that fails to parse as valid JSON at all - which is why the result wasn't a truncated report, it was nothing. `null`, indistinguishable from "the field was never populated."

The earlier tool calls in the sequence - a search query, a lookup by ID - needed very little output to construct. They worked fine, every time, because they never came close to hitting the ceiling. The bug only showed up on the call that needed a real output budget, which is exactly why it looked like something specific to that one method, rather than a global configuration gap.

Editing this setting in the application.yml file:
```
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY:NONE}
      chat:
        options:
          model: claude-haiku-4-5
          max-tokens: 4096
```
and setting `max-tokens` explicitly, generously, fixed it completely.

## What actually mattered

None of my first three hypotheses were unreasonable, given what I could see at each point - they were reasonable next questions to ask given what I could see.  What was useful was:
- Simplifying the tool down to a single parameter so that I could see the structure of the data being output by Claude (or lack of output by Claude as the case turned out).
- Being able to see how the tool was actually being invoked.

The first three calls worked fine, but told me nothing useful about whether the fourth one would - a 500-token default is plenty for "here's a search query," and nowhere close to enough for "here's a multi-paragraph synthesized report." Same limit, wildly different amount of room it actually needed to leave.

My takeaway from this is that **token limits don't fail loudly**.  There's not necessarily a clear "response truncated" exception; it just quietly produces bad output masquerading as something completely different.  The token budget has to take into account the most demanding query that could be executed, without simply going with `Integer.MAX_VALUE` (because that is going to end badly).  A good tool should provide insight into token usage, something I hadn't designed before but am working on providing.
