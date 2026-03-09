---
layout: post
title: "What Dead Code Taught Us About Building Tools for AI Agents"
tagline: "Graphs are primitives. Dead code removal is just the first application."
description: "How we benchmarked AI agents on dead code detection across 40+ real-world codebases, what we learned about context engineering, and why code graphs are the missing primitive for AI-powered development tools."
category: engineering
tags: [supermodel, dead-code, ai-agents, static-analysis, code-graphs, benchmarks]
---

# What Dead Code Taught Us About Building Tools for AI Agents

*We set out to build a dead code finder. We ended up discovering something bigger: code graphs are the foundational primitive that AI-powered development tools are missing.*

---

## The Moment That Changed Our Thinking

We asked an AI agent to document a codebase. It did a thorough job -- read the source files, traced the patterns, and produced clean architectural docs. There was just one problem: a third of the features it documented didn't exist anymore.

The functions were still in the code. They had clear names, proper signatures, even JSDoc comments. But they hadn't been called in months. They were remnants of past refactors, stranded by pattern drift, invisible to the agent because nothing in the source files said "ignore me, I'm dead."

This is the dead code problem, and it gets worse with AI-generated code. When engineers write code by hand, they develop intuition for what's alive. They remember the refactor that replaced the old auth flow. They notice when editing a method produces no visible change -- that sinking feeling of realizing the method was dead all along. But AI agents have no such memory. They see every function as equally real.

In vibe-coded software -- especially after multiple refactors -- dead code accumulates fast. It clogs context windows, confuses agents, and wastes the most expensive resource in AI-powered development: tokens spent reasoning about code that doesn't matter.

We decided to do something about it.

---

## The Thesis: Graphs Are Primitives

When we started building Supermodel, we had a hypothesis: **if we build good core graph tools -- call graphs, dependency graphs, domain models -- we could provide a platform for building an entire category of static analysis and code quality tools on top.**

Dead code detection was our first test case. Not because it's the hardest problem, but because it sits at the intersection of everything we care about: it requires understanding relationships between symbols, it directly impacts AI agent effectiveness, and it has a clear ground truth (either something is called or it isn't).

The technical insight was straightforward. With a well-constructed call graph and dependency graph, you can identify *dead code candidates* -- symbols that have no inbound references. Naively, that means: "anything that is not imported or not called is probably dead."

But "probably" is doing a lot of work in that sentence.

---

## Why Naive Reachability Isn't Enough

Static analysis can trace imports and function calls. What it can't easily see are the boundaries of indirection that make code *appear* dead when it isn't:

**Framework entry points.** A Next.js `page.tsx`, a NestJS `@Controller()`, an Express route handler -- none of these are "called" by your code. They're invoked by the framework at runtime. A naive dead code detector would flag every API endpoint as unused.

**Event-driven and plugin architectures.** Webhook handlers, message queue consumers, dynamically loaded plugins -- all registered through patterns that static analysis struggles to trace.

**API boundaries.** When a service exposes functions through a REST or GraphQL API, the callers live on the other side of a network boundary. The server-side handler has zero internal callers, but it's the most critical code in the system.

**Generated code patterns.** Code generators (ORMs, gRPC stubs, GraphQL codegen) produce symbols that aren't called until the rest of the system is wired up. These often live in conventionally-named directories like `generated/`, `target/`, or `__generated__/`.

**Re-exports and type-level usage.** A type that's re-exported through a barrel file (`index.ts`), or a constant used only in type annotations -- these are alive but invisible to call-graph-only analysis.

These aren't edge cases. In a typical production codebase, they represent 30-60% of all exported symbols. Flag them all as dead and you've built a tool nobody trusts.

---

## Our Approach: Probabilistic Candidates + Agent Verification

Instead of trying to build a perfect static analyzer (an impossible task), we designed a system that works *with* AI agents rather than replacing them.

**Step 1: Graph Analysis.** Parse the codebase with tree-sitter. Build the call graph and dependency graph. Run BFS reachability from identified entry points (framework conventions, main files, test files). Everything unreachable becomes a candidate.

**Step 2: Probabilistic Ranking.** Not all candidates are equally likely to be dead. We rank by signals: Is it in a generated directory? Does it follow a framework naming convention? Is it a type re-export? How deep is it in the import chain? This produces a ranked list of candidates, from "almost certainly dead" to "suspicious but uncertain."

**Step 3: Agent Verification.** Hand the ranked candidates to an AI agent. The agent can read surrounding code, check for dynamic usage patterns, and apply judgment that static analysis can't. The key insight: **the agent's job is now filtering a short list, not searching an entire codebase.** This is dramatically more tractable.

**Step 4: Learn and Refine.** Track which candidates turn out to be false positives. Learn that projects using Next.js have `page.tsx` files that look dead but aren't. Learn that `__mocks__/` directories are test infrastructure. Feed this back into the ranking model.

The cumulative effect: each iteration produces fewer false positives for the agent to sort through, the verification gets faster and cheaper, and the system builds institutional knowledge about project patterns.

---

## Benchmarking: How We Measured

We used [mcpbr](https://github.com/greynewell/mcpbr) (Model Context Protocol Benchmark Runner) to run controlled experiments. The setup:

- **Model**: Claude Sonnet 4 via the Anthropic API
- **Agent harness**: Claude Code
- **Two conditions**: (A) Agent with Supermodel graph analysis pre-computed, (B) Baseline agent with only grep, glob, and file reads
- **Same prompt, same tools** (minus the analysis file), same evaluation

### Ground Truth: How Do You Know What's Actually Dead?

This is the hardest part of benchmarking dead code detection. You need to know -- with certainty -- which symbols in a codebase are dead. We used two approaches:

**Synthetic codebases.** We built a 35-file TypeScript Express app and intentionally planted 102 dead code items: legacy integrations, deprecated auth methods, feature flags that were never cleaned up, replaced utility functions. We know exactly what's dead because we put it there. This is useful for development but doesn't reflect real-world complexity.

**Real pull requests from open-source projects.** This is where the benchmark gets interesting. We searched GitHub for merged PRs whose commit messages and descriptions explicitly mention removing dead code, unused functions, or deprecated features. The logic: if a developer identified code as dead, removed it in a PR, the tests still pass, and the PR was approved by reviewers and merged -- that's confirmed dead code.

For each PR, we extracted ground truth by parsing the diff: every exported function, class, interface, constant, or type that was *deleted* (not moved or renamed) became a ground truth item. The agent's job is to identify these same items by analyzing the codebase at the commit *before* the PR -- the state where the dead code still exists.

This methodology has a key strength: it's grounded in real engineering decisions, not synthetic judgment calls. A human developer, with full context of the project, decided this code was dead. We're asking: can an AI agent reach the same conclusion?

We tested against PRs from 12 repositories:

| Repository | Stars | PR | What was removed | Ground Truth Items |
|-----------|-------|-----|-----------------|-------------------|
| antiwork/Helper | 655 | [#527](https://github.com/antiwork/helper/pull/527) | Unused components and utils | 12 |
| Grafana | 66K | [#115471](https://github.com/grafana/grafana/pull/115471) | Drilldown Investigations app | 42 |
| Next.js | 138K | [#87149](https://github.com/vercel/next.js/pull/87149) | Old router reducer functions | 17 |
| Directus | 29K | [#26311](https://github.com/directus/directus/pull/26311) | Deprecated webhooks | 15 |
| Cal.com | 40K | [#26222](https://github.com/calcom/cal.com/pull/26222) | Unused UI components | 8 |
| Maskbook | 1.6K | #12361 | Unused Web3 hooks and contracts | 22 |
| OpenTelemetry JS | 3.3K | #5444 | Obsolete utility functions | 4 |
| Logto | 11.6K | #7631 | Unused auth status page | 8 |
| jsLPSolver | 449 | #159 | Dead pseudo-cost branching code | 10 |
| Mimir | -- | #3613 | Unused functions and types | 9 |
| tyr (track-your-regions) | -- | #258 | Dead image service and auth code | 22 |
| Podman Desktop | -- | #16084 | Unused exports | 2 |

Across 40+ benchmark runs, we evaluated both agents on precision (what fraction of reported items are actually dead), recall (what fraction of actually dead items were found), and F1 score.

---

## Results

### The Headline: A Real Production Codebase

Our strongest result came from [antiwork/Helper](https://github.com/antiwork/helper), a production Next.js application with 576 files and 1,585 declarations. Ground truth: 12 dead code items confirmed by a [merged PR](https://github.com/antiwork/helper/pull/527) where a developer identified and removed them. We ran this test twice to confirm reproducibility.

| Metric | With Graph | Baseline (grep) | Improvement |
|--------|-----------|-----------------|-------------|
| **Recall** | **91.7%** (11/12) | 16.7% (2/12) | **5.5x** |
| **Precision** | **100%** | 100% | -- |
| **F1** | **95.7%** | 28.6% | **3.3x** |
| **Tool Calls** | 6 | 184 | **30x fewer** |
| **Runtime** | 33-42s | 343-792s | **8-24x faster** |
| **Cost** | $0.10-0.11 | $2.64-8.61 | **24-86x cheaper** |

The graph-enhanced agent found 11 of 12 confirmed dead code items with zero false positives. The baseline found 2. Both runs reproduced identically on recall and precision. **This task was "resolved"** -- both precision and recall above our 80% bar -- making it the only real-world task where any agent cleared that threshold.

What happened? The baseline agent spent 184 tool calls grepping through 576 files trying to build a mental model of the call graph at runtime. The graph-enhanced agent read one JSON file, wrote a small Python script to extract the candidates, and was done in 6 tool calls. **The graph pre-computes the expensive work, so the agent doesn't have to.**

The single missed item (`SUBSCRIPTION_FREE_TRIAL_USAGE_LIMIT`) was a constant used only in template literals -- a known gap in the parser, not a limitation of the approach.

### Synthetic Codebases: Near-Perfect

On a synthetic 35-file TypeScript Express app with 102 planted dead code items:

| Metric | With Graph | Baseline |
|--------|-----------|----------|
| **Recall** | **99%** | 39% |
| **Precision** | 91% | 100% |
| **F1** | **95%** | 56% |
| **Tool Calls** | 4 | 68 |

The baseline achieves perfect precision by being conservative -- it only reports what it's absolutely sure about, and misses 61% of dead code. The graph-enhanced agent finds nearly everything.

### Scaling to Larger Repositories: High Recall, Precision Work Ahead

We then ran the graph agent against real PRs from 12 open-source repositories. The pattern was consistent: **the graph agent found most of the dead code, while the baseline agent found almost nothing.**

In our most controlled comparison (Feb 20 run, 10 real-world tasks, identical conditions), the results were stark:

| Real-World Task | Graph Recall | Baseline Recall |
|-----------------|-------------|-----------------|
| Directus (29K stars) | **100%** (15/15) | 0% |
| Podman Desktop | **100%** (2/2) | 0% |
| Latitude LLM | **100%** (5/5) | 0% |
| tyr_pr258 | **95.5%** (21/22) | 0% |
| Mimir (Statistics Norway) | **77.8%** (7/9) | 0% |
| OpenTelemetry JS | **75.0%** (3/4) | 0% |
| Maskbook (Web3) | **63.6%** (14/22) | 0% |
| jsLPSolver | 50.0% (5/10) | 0% |
| Gemini CLI | 33.3% (2/6) | 0% |
| Logto | 0% (0/8) | 0% |

**The baseline scored 0% recall on every single task.** The grep-based approach -- even with 30 iterations and unlimited tool calls -- couldn't find any confirmed dead code across these codebases. The graph agent, by contrast, achieved 75%+ recall on 6 of 10 tasks.

But here's what we need to be honest about: **precision is still the challenge.** The graph agent's high recall came with high false positive counts. When the agent found all 15 Directus items, it also reported 2,450 false positives. On Maskbook, 14 true positives came alongside 5,880 false positives. The agent was reading the analysis file and transcribing candidates rather than filtering them -- an "analysis dump" pattern we're actively working to solve.

Across all head-to-head matchups (16 runs with both agents):

| Metric | Graph Agent | Baseline Agent |
|--------|------------|----------------|
| Head-to-head wins | **9** | 2 |
| Ties | 5 | 5 |
| Resolved (P>=80%, R>=80%) | **1** (antiwork/helper) | 0 |

---

## Three Failure Modes We Discovered

### 1. The File Size Wall (Fixable)

Large analysis files exceed tool output limits. A 6,000-candidate analysis exceeds the 25K token tool output limit, so the agent either gets a truncated view or errors out.

**Fix**: Split candidate lists into chunks of 800 entries with a manifest file. This took Maskbook recall from 0% to 64% overnight.

### 2. API Recall Gaps (Fixable at the Parser Level)

Sometimes the Supermodel parser misses ground truth items entirely. The Logto benchmark found 0 of 8 ground truth items in the analysis -- no amount of agent intelligence can find what the analysis doesn't contain.

Root causes we've identified: `export default` not tracked, type re-exports (`export type { X } from`) missed, test file imports not scanned. These are being fixed systematically.

### 3. Agent Non-Determinism (Partially Addressable)

Same task, same config, different results. One run finds 3 true positives; the rerun finds 0. The only fully deterministic path was pre-computed analysis on small codebases, where the agent reads a file and transcribes it.

This is an inherent property of LLM-based agents. The mitigation is to reduce the agent's degrees of freedom: give it a shorter, better-ranked candidate list so there's less room for the agent to go off-track.

---

## The Scaling Insight

This is the finding we keep coming back to:

| Codebase | Files | Baseline Recall | Graph Recall | Baseline Cost | Graph Cost |
|----------|-------|-----------------|--------------|---------------|------------|
| Synthetic Express app | 35 | 39% | 99% | $0.79 | $0.40 |
| antiwork/Helper | 576 | 17% | **92%** | $2.64-8.61 | $0.10-0.11 |
| Mimir (Statistics Norway) | 351 | 0% | **78%** | $0.62 | $0.22 |
| Directus | ~2,000 | 0% | **100%** | $1.03 | $0.16 |
| Maskbook (Web3) | 2,899 | 0% | **64%** | $0.47 | $0.21 |

As codebases grow:
- **Baseline recall collapses to zero** -- the search space overwhelms the agent completely
- **Baseline cost increases** -- more files means more tool calls spent finding nothing
- **Graph recall stays high** -- pre-computed relationships don't scale with file count
- **Graph cost stays flat** -- the agent reads one analysis file regardless of codebase size

The graph absorbs the complexity that would otherwise land on the agent. This is the fundamental value proposition, and it applies to any tool built on graph primitives, not just dead code detection.

---

## Lessons for Building AI-Powered Code Tools

### 1. Context engineering matters more than model capability

Same model, same tools, different input structure: 5.5x better recall. The model wasn't the bottleneck -- the signal-to-noise ratio of its input was.

This is the core lesson. **Good prompting is high-signal prompting.** The best thing you can do for an AI agent isn't give it a smarter model -- it's give it pre-computed, structured, relevant context and eliminate the noise.

### 2. Pre-compute what you can, delegate judgment to the agent

Static analysis is good at exhaustive enumeration. AI agents are good at judgment calls. The worst outcome is making the agent do both: enumerate *and* judge. That's 184 tool calls and 17% recall.

The best outcome is a pipeline: graphs enumerate candidates, agents verify them. Each component does what it's best at.

### 3. Precision is harder than recall (and matters more for trust)

Our graph agent consistently achieved high recall on real codebases -- it found the dead code. But it also reported thousands of false positives. A tool that says "here are 3,000 things that might be dead" isn't useful. A tool that says "here are 15 things that are dead, and here's why" is.

The precision problem is solvable through better ranking, better framework-aware filtering, and learning from false positive patterns. This is active work.

### 4. Real-world codebases are dramatically harder than synthetic ones

On our synthetic benchmark, we hit 95% F1. On a well-structured 576-file production app, we resolved the task with 95.7% F1. But on large monorepos (80MB+), recall stays high while precision collapses -- the agent finds the dead code but can't filter the false positives yet. Synthetic benchmarks are necessary for development but insufficient for evaluation. You need both.

### 5. The system improves iteratively

Every benchmark run teaches us something:
- jsLPSolver taught us that well-organized small repos favor grep-based search
- Maskbook taught us about the file size wall
- Logto taught us about parser gaps in `export default`
- Directus taught us about the analysis-dump failure mode

Each lesson feeds back into the parser, the ranking model, and the agent prompt. The system gets better with each iteration -- not through model improvements, but through better context engineering.

---

## The Bigger Picture: Graphs as Factory Primitives

Dead code detection is one application. But the underlying primitive -- a structured graph of code relationships -- enables an entire category of tools:

- **Impact analysis**: "If I change this function, what breaks?" (call graph)
- **Architecture documentation**: "What are the domains and boundaries in this system?" (domain graph)
- **Dependency auditing**: "Which packages are actually used?" (dependency graph)
- **Refactoring assistance**: "Show me all the callers of this deprecated API" (call graph)
- **Security surface mapping**: "What code paths lead from user input to database queries?" (call graph + data flow)

Each of these has the same structure: pre-compute the graph, rank candidates, let agents handle judgment. The graph is the primitive. The applications are built on top.

---

## Try It

The Supermodel API is available today. Generate a dead code analysis for your codebase:

```bash
# Create a repo archive
cd /path/to/repo
git archive -o /tmp/repo.zip HEAD

# Analyze (via the dead code endpoint)
curl -X POST "https://api.supermodeltools.com/v1/analysis/dead-code" \
  -H "X-Api-Key: $SUPERMODEL_API_KEY" \
  -H "Idempotency-Key: $(git rev-parse --short HEAD)" \
  -F "file=@/tmp/repo.zip"
```

Or use the [Supermodel MCP server](https://github.com/supermodeltools/mcp) to give your AI agent direct access to graph analysis in real time.

The graph endpoints (call graph, dependency graph, domain graph, parse graph) are all available through the same API. If you see a different application for graph primitives -- impact analysis, architecture visualization, security auditing, or something we haven't thought of -- we'd love to hear about it. We're building the raw materials. What you build with them is up to you.

---

## Methodology Notes

- **Benchmark framework**: [mcpbr](https://github.com/greynewell/mcpbr) v0.13.4
- **Model**: Claude Sonnet 4 (`claude-sonnet-4-20250514`)
- **Agent harness**: Claude Code
- **Total benchmark runs**: 40+
- **Total cost**: ~$75 across all runs
- **Repositories tested**: 12 open-source projects (29K-138K GitHub stars)
- **Ground truth sources**: Synthetic corpus (hand-curated) + merged PRs with passing CI
- **All runs logged** with timestamps, configs, full agent transcripts, and structured metrics
- **Analysis engine**: Supermodel (tree-sitter-based parsing, BFS reachability analysis)

---

*Supermodel provides code graph APIs for AI-powered development tools. [Get started with the API](https://docs.supermodeltools.com) or [try the MCP server](https://github.com/supermodeltools/mcp).*
