---
layout: post
title: "How AIOps Systems Should Evolve with Software: Lessons from CloudMate"
date: 2026-01-03
---

In the previous article, we reviewed the current state of the art in AIOps. From the angle of performance troubleshooting, Context Engineering has emerged as the core methodology in AI-driven operations. The central idea is direct: collect system state, historical experience, and expert knowledge in a structured way, giving large language models enough context to understand problems and produce solutions the way a seasoned engineer would. Current system state combined with a stable historical knowledge base can unlock substantial LLM capability.

Intelligent operations depend heavily on knowledge bases. RCACopilot is a clear example: its effectiveness depends on the large root-cause analysis database accumulated over years of Microsoft's mail service. Without that database, performance drops by orders of magnitude, making the system nearly unusable. Yet nearly every disclosed knowledge-base-driven system ignores a foundational software engineering reality: software changes. In fast-moving systems, documentation goes stale and knowledge goes stale, which creates a persistent challenge for any knowledge-base-driven AIOps system.

The challenge has two dimensions. First, how do you keep the knowledge base current with rapidly evolving code? Second, how does software evolution avoid breaking the effectiveness of AI-driven operations?

Tencent Cloud's CloudMate is the first system to make a systematic attempt at both dimensions.

This article is based on CloudMate's presentation at GOPS Shanghai 2025. Because the technical details disclosed were limited, the focus here is on high-level design and the insights it offers.

## The Dead End of Self-Updating Knowledge Bases

The idea that a knowledge base should update itself automatically is not new.

Long before AIOps existed, many teams recognized the limits of manually maintained knowledge bases. Teams have repeatedly proposed self-evolving knowledge systems, trying to keep documentation in sync with software. After LLMs arrived, the rise of RAG brought another wave of enthusiasm for dynamic knowledge-base updates. Many teams announced intentions; very few shipped working results. The fundamental reason: validating knowledge updates is genuinely hard. Quality is difficult to quantify, and large volumes of uncontrolled updates tend to push the knowledge base into an unmanageable state.

Consider a typical scenario: a service split replaces the monolithic monitoring API `/metrics/query` with `/metrics/service/query` and `/metrics/infrastructure/query`. An engineer updates the API documentation and adds the new usage to the knowledge base.

This single update has consequences that are hard to track across the rest of the knowledge base. The old API reference in a CPU anomaly runbook now returns a 404. A slow-service diagnosis workflow assumes unified monitoring data and now requires querying two endpoints separately. All example code in an alerting configuration guide written a year ago is invalid.

Each new document added creates potential conflicts that grow at O(n) with the size of the knowledge base. A new document can produce semantic conflicts, interface conflicts, or assumption conflicts with any existing document. Verifying those conflicts requires understanding the context, dependencies, and implicit assumptions of every document involved. That exceeds the capacity of manual maintenance.

There is a second dimension to this problem. From the perspective of the software being operated, the operability of the system itself is equally hard to assess. Principles like shift-left operations and observability exist, but for any specific system it is difficult to judge whether it is agent-friendly or suitable for automated operations. As software evolves continuously, how do you ensure each change does not degrade agent operability?

This is the most important distinction between CloudMate and existing products, and the core question this article examines: how to build stable, reliable AI operations capability in a dynamically evolving environment.

## CloudMate's Approach

CloudMate's core reasoning: since validating the quality of the knowledge base itself is difficult, validate the end result directly. This approach addresses both dimensions of iteration stability at once. Updates to the software system cannot break agent operability. Updates to the knowledge base cannot reduce the agent's problem-solving capability.

CloudMate uses a two-layer architecture that separates online troubleshooting from offline knowledge evolution.

**Top layer: online troubleshooting system**

This is a conventional agent loop with three core components. The **knowledge base** stores domain knowledge the agent needs to execute tasks: business topology, failure pattern libraries, expert diagnostic logic, and standardized troubleshooting procedures. The **agent loop** is a classic plan-execute-observe cycle where the agent forms a troubleshooting plan guided by the knowledge base, calls tools, observes results, and iterates. The **tool library** provides the actual troubleshooting capabilities: log tools, metrics tools, database tools, and others.

**Bottom layer: offline validation and evolution system**

This is CloudMate's core innovation. It rests on a simple principle: **the best validation environment is practice itself**. Whether knowledge is effective is most reliably determined by using it in real tasks. The system pursues this through three components in a sandbox environment.

The **sandbox** provides an isolated validation space, ensuring knowledge updates cannot affect production services. The **case library** stores the complete trajectories (plan-execute-observe) generated during historical task execution, including the agent's reasoning, the tools called, and the results returned. These trajectories serve two purposes: as baseline cases for knowledge evolution, and as regression tests for system operability.

The **exploration loop** is the critical mechanism. The CloudMate team defines baseline cases for different scenarios, each with explicit, verifiable completion criteria. The system combines objective metrics (such as time elapsed and number of tool calls) with a supervisor model's scores to measure solution quality. The process: multiple agents retry baseline cases in parallel within the sandbox; the scoring system summarizes new knowledge from successful and failed attempts; that knowledge is not committed directly but triggers re-validation against the baseline cases; updates are accepted only when performance reaches the same level or better. This ensures every iteration of the knowledge base is an improvement.

The historical trajectories in the case library serve a parallel purpose. When the operated system pushes a change, those trajectories are re-executed using the current system state to obtain tool call results. If the results differ significantly from the historical record, the change may have broken system operability and warrants investigation. That could mean updating documentation or reconsidering the change itself.

The full system creates two guarantees: the exploration loop ensures knowledge base evolution does not regress, and the case library ensures system evolution does not break operability.

### What the Design Reveals

Does this system actually work?

The presentation did not disclose specific performance data: knowledge base update acceptance rates, number of issues caught by the case library, or measured improvement in agent capability. Even without quantitative metrics, the design itself makes the core reasoning visible.

Returning to the two challenges raised at the start: how does the knowledge base keep up with code evolution? CloudMate uses the exploration loop to update the knowledge base automatically and uses baseline cases to verify that capability does not decrease after each update. How does software evolution avoid breaking operability? CloudMate uses the case library to treat historical execution trajectories as regression tests, catching breaking changes early.

The exploration loop asks: after using this knowledge, can the agent complete the baseline task? The case library asks: after this change, does the agent's execution trajectory show abnormal deviation? Both mechanisms focus on validating the end result.

## Implications for Practice

The conventional approach tries to validate the correctness of knowledge: is this document accurate, does this diagnostic logic follow the standard, does this procedure have gaps? Correctness is subjective, hard to quantify, and context-dependent.

CloudMate shifts the question: regardless of what the knowledge base contains, the only thing that matters is whether the agent can ultimately solve the problem. This represents a paradigm shift from validating the input (knowledge) to validating the output (capability).

This reasoning is well established in software engineering. We validate system capability through tests, benchmarks, and load tests rather than arguing about the theoretical correctness of code, algorithms, or architecture.

CloudMate applies the same engineering philosophy to AI systems: define capability standards with baseline cases, verify achievement with automated testing, prevent regression with regression tests. This gives the fuzzy concept of knowledge quality an operational measure. The question changes from whether a piece of knowledge is good to whether agent capability improves after using it. The first question has no answer. The second is testable.

This is one of the AI-Native principles the industry repeatedly mentions but rarely articulates clearly: build quality assurance for AI systems on verifiable capability outputs rather than knowledge inputs that resist quantification.

On architecture, CloudMate uses a two-library separation strategy, keeping what the system should know (knowledge base) apart from what it can actually do (case library). The analogy in software engineering is specifications versus test cases. The knowledge base defines expectations; the case library validates reality. This represents a fundamental shift in how agents are treated: they require the same rigorous management and quality assurance as APIs or databases.

On process integration, CloudMate embeds agent capability management into DevOps. The CI/CD pipeline includes agent capability validation, knowledge base synchronization, and capability regression tests. The monitoring system tracks agent capability health (baseline task pass rate) and knowledge base freshness (the time gap between knowledge updates and system changes).

On quality assurance, CloudMate treats continuous validation as required infrastructure. In a dynamic environment, continuous validation moves beyond being an optional cost. It is the necessary mechanism for ensuring evolution does not regress.

## Open Questions

CloudMate demonstrates one viable path toward AI system self-evolution. Because the presentation provided limited information, the details of several key mechanisms were not disclosed.

### The Evaluation Paradox

CloudMate uses a combined evaluation framework of objective metrics plus supervisor model scores. Objective metrics (such as time elapsed and tool call count) are relatively reliable. Supervisor model scores introduce a more fundamental paradox.

CloudMate's core insight is to validate capability rather than knowledge, because assessing the quality of knowledge itself is hard. But within the exploration loop, the supervisor model is essentially trying to assess whether a trajectory is good: whether the agent's execution process, reasoning path, and tool choices are appropriate. This faces the same difficulty as assessing knowledge quality. If we already knew what a good trajectory looks like, there would be no need for an LLM. We could write rules directly.

This paradox points to the core problem in evaluation: we use LLMs because problems are too complex for humans to define what a good solution looks like, yet validating LLM output requires another model to define what a good solution looks like. Where do the supervisor model's standards come from? How does it avoid introducing new biases? How do those standards update as the business evolves?

The presentation did not disclose how CloudMate handles this paradox. These details are essential for understanding the reliability of the system.

### The Limits of Validation

The sandbox is the foundation of validation, but replayability has limits. Technically, intermittent failures caused by concurrent race conditions, problems that depend on real user behavior, and complex interactions in distributed systems are difficult to reproduce faithfully. From a safety perspective, scenarios involving user private data and troubleshooting workflows that include high-risk operations are not suitable for sandbox replay.

These boundaries directly constrain the completeness of validation. For scenarios that fall outside them, human review or supplementary methods may still be required, which constrains the degree of automation achievable.

### The Limits of Automation

Even with most of the process automated, removing human involvement entirely may be neither realistic nor desirable. Some situations likely still require human intervention: abnormal fluctuations in evaluation metrics, consecutive sandbox test failures, and significant changes in the external environment.

The key question is how to identify the threshold at which human intervention is needed and how to balance automation with controllability.

CloudMate demonstrates the possibility of AI system self-evolution and makes visible the fundamental challenges: the credibility of evaluation mechanisms, the limits of validation capability, and the balance between automation and control. Those of us working in this space need to explore answers in our own practice.
