---
layout: post
title: "Agent Management: The Deterministic Boundaries of AI Coding"
date: 2026-03-07
---

Over the past year, we have been in the trenches with hundreds of developers in the Agent Management forum, running experiments, doing postmortems, and iterating. This article is our interim summary.

## I. A Debate That's Over

"Code written by AI is simply unusable."

People have been saying this since ChatGPT appeared at the end of 2023, and they kept saying it through 2025. AI kept raising the ceiling on what it could handle, but the criticism never stopped.

Until December 2025. Andrej Karpathy declared that AI programming would define the future of software engineering. Linus Torvalds started writing code with AI. Even the most stubborn old-school hackers began to concede.

Meanwhile: Anthropic used Claude Code to write a C compiler from scratch that can compile the Linux kernel. A fully AI-written, scalable file system appeared at the FAST conference. PingCAP CTO Huang Dongxu and Taichi Graphics founder Hu Yuanming are both deep users of AI Coding and are actively rolling it out across their teams.

So the question becomes: with the same AI, why do some people build production-grade systems and scale them across organizations, while others cannot get a personal web project to work without holes everywhere?

The short answer: AI Coding is a management problem, not a technology problem.

## II. The Frustrated Technical Pioneer

Anthropic published a C compiler built entirely with Claude Code that successfully compiled the Linux 6.9 kernel. Beyond the hype from marketing accounts, the skepticism came mostly from the technical community: macros go unused, paths are hardcoded, there is no handling of x86 16-bit real-mode boot code. And so the comment threads end with the usual conclusion: "See? AI still can't do it. I'm still irreplaceable."

The more technically accomplished someone is in the traditional sense, the slower they tend to adopt AI Coding. The reason has nothing to do with the AI. Their management thinking has not caught up.

**AI as Colleague**

Imagine you hired someone with exceptional intelligence and learning ability who is on their first day of work. You tell them: "Write a compiler that can compile Linux." One sentence, nothing else. What happens?

They will not produce a fully-featured compiler compatible with every ecosystem. Can it handle Hello World? You did not say. Dynamic linking? You did not mention it. Does the 16-bit real mode need to be implemented from scratch, or should it use existing tools? When you review the work, you call it a mess. The problem is your management.

Your intuition says a compiler "should" correctly compile all valid C code. Someone else's intuition says a compiler can be a course project that handles basic syntax. Both are reasonable definitions. AI will meet whatever standard you define, then stop. If your standard is vague, it will stop where common sense lands, and that common sense may not be yours.

This explains why technical experts resist. They are skilled at translating requirements into precise code. The problem sits upstream: stating requirements clearly, drawing explicit boundaries, quantifying acceptance criteria. The experts who mock the Claude Code compiler forget that the good software they have shipped in the past succeeded not because of them alone, but because of clear product definitions.

**A Century-Old Problem**

Economics has studied a classic problem for decades called the **principal-agent problem**. The word "agent" in economics and the AI Agent we talk about today are the same word. AI is now capable enough to be treated as an agent in the economic sense.

When you delegate a task to an agent, three challenges arise naturally: **information asymmetry** (you do not fully know what it is doing), **goal misalignment** (its interpretation diverges from your intent), and **verification difficulty** (you struggle to judge whether the output is actually good).

Management theory's answer is to build contracts and structures: clear goal definitions, process checkpoints, measurable acceptance criteria. Our answer is structurally identical. This is why the forum is named "Agent Management."

## III. Deterministic Boundaries

Just as a team leader has no time to review every line of code and must trust their colleagues' competence, good management sets clear standards at key nodes and leaves space in between. We call these key nodes **deterministic boundaries**. The entire AI Coding workflow breaks into three segments, each with an explicit contract.

### 3.1 Requirements: Documents Are the Artifact, AI Is the Compiler

In traditional development, code is the final deliverable. In the AI Coding era, documentation is.

AI acts more like a compiler: it compiles your documentation into runnable code. Output quality depends on input quality. This means documentation must be solid. Our practice: start from the PRD, generate an architecture document, refine it into detailed execution documents, then have AI implement strictly from those execution documents.

Once documentation becomes the primary artifact, the documentation itself needs acceptance testing. Generating a spec from one sentence produces the same result as generating software from one sentence. Both hand the uncertainty to AI, and both will eventually drift out of control.

Two difficulties arise here. First, AI generates documents tens of times faster than humans can read them. Second, asking AI to review documentation yields responses like "structure is clear, recommend adding exception handling." Correct, useless, nothing resolved.

The real issue is that most people, when handed a spec, do not know what they should actually be looking at. Many engineers instinctively audit architecture soundness, interface naming conventions, data model elegance. AI performs well on all of these.

The only thing that actually requires human review is **user stories**.

User stories describe real people doing real things with software. "As a streamer, I need to start a poll in my live room." That is a user story. It does not care about your architecture choices, API naming, or database selection. It cares about one thing: can this person accomplish what they need to do.

Turn that around: if your documentation allows every user story to walk end-to-end without gaps, where every required interface is defined, every parameter and return value is complete, and every exception is covered, the documentation is complete.

This is what we call **document testing**: using user stories as test cases to run against the documentation. Have AI walk through each user story step by step. The result is deterministic and depends on no one's judgment.

Architecture-level review still matters. It just does not require a human. Cross-validation between different models from different angles produces better results at lower cost than one person staring at the document.

Humans own user stories. AI owns technical details. Each stays in its lane.

### 3.2 Execution: The Boundary of Attention

AI shares a characteristic with humans: **attention is finite**. When the context window is overloaded, performance drops significantly. Every task handed to AI must stay within what it can handle. Two dimensions apply.

**Spatially: modularity.** Provide complete input conditions: what this module does, what the inputs are, what the outputs are. Write the API contract clearly. Frame the boundaries with a test suite. Within that defined boundary AI can work freely. Everything outside it is not its concern.

**Temporally: incremental steps.** Even within a single module, you cannot dump all requirements at once and hope for the best. Break the task into small pieces, accept each piece before moving to the next, and reset context for each new piece.

The cleaner and more focused the context you provide, the higher the output quality. On top of these two dimensions, we run an independent AI to review each commit, verifying that changes stay within the scope of the current task.

### 3.3 Acceptance: No Standard, No Complaint

This is the most important of the three segments.

Most dissatisfaction with AI Coding traces back to unclear acceptance criteria. Back to the compiler case: why was the technical community's criticism beside the point? Because the acceptance criteria were "compile Linux," not "build a general-purpose compiler." Whatever you did not write into the contract, the agent has no obligation to deliver.

AI produces output at one hundred times the speed of a human. At that speed, a small deviation left uncaught for a few hours becomes a systemic problem across dozens of files. You need to verify immediately after each step is complete.

Our approach is to **front-load the testing infrastructure**. After all execution documents are generated and before the first line of business code lands, build a complete test framework based on the API contracts. All dependencies that do not yet exist get replaced by mock servers and mock data, creating an isolated test system decoupled from any live service. When AI drifts, the tests tell you immediately.

If you find you cannot write tests, step back before writing code. The inability to write tests signals that the requirements were ambiguous from the start. Return to the documentation.

Beyond testing, **adversarial code review** provides another layer. After each AI output, hand it to a different AI (ideally a different model) to audit: does this strictly follow the execution document? The test infrastructure verifies whether results are correct. Adversarial review verifies whether the process stayed within bounds. Together they form the deterministic boundary at the acceptance stage.

## IV. The AI-Native Team

Chapter III covers how to manage one agent doing one thing. In practice, you will not run a single session. Among forum members, simultaneously orchestrating twenty to thirty agents working in parallel is now routine.

When running multiple agents in parallel, the infrastructure from Chapter III — API contracts, mock servers, and test suites — shifts from good practice to survival requirement. We learned this the hard way: each agent's individual output passed its own tests, but when combined, conflicts appeared everywhere because the contracts between modules were not tight enough. Git worktree isolation, now standard in both Cursor and Claude Code, addresses the environment side of this problem.

The technical coordination problem has a solution. Once solved, a more fundamental question surfaces: **how should people organize?**

One person managing thirty agents produces output that exceeds what a traditional team of five to ten engineers delivers. When productivity changes at this scale, organization must follow. Steam engines turned workshops into factories, not because anyone decided factories were better, but because workshops could not contain the new capacity. The traditional structure of one tech lead and five to six engineers generates more coordination overhead than value when everyone can run thirty agents.

Two models have emerged from what we observe.

**The Alpha Model**

PingCAP CTO Huang Dongxu demonstrated the power of the alpha model in his Vibe Engineering practice: a single engineer driving large-scale agent collaboration, rewriting TiDB's PostgreSQL compatibility layer with AI, achieving near-production code quality. His analogy is precise: the alpha wolf cultivates the pack's territory.

The advantages are clear: decision chains are minimal, communication overhead is zero, one person's judgment runs through the entire product. Two hard constraints exist, though.

First, the same territory rarely accommodates two alphas. Elite engineers each build highly personalized agent workflows, prompting styles, and coding conventions. Two top engineers struggle to collaborate on the same module because their respective agent teams speak different languages.

Second, the ceiling is acceptance throughput. Dongxu himself has said he spends ninety percent of his time evaluating AI output. Someone with his capability can sustain that. For most people, the agent army produces at one hundred times human speed, and when acceptance cannot keep pace, technical debt compounds faster than it accumulates through ordinary means.

The alpha model proves AI-native development is viable. The bar for individuals is very high.

**Breaking Down the Super-Individual**

Since super-individuals are hard to find, decompose the super-individual's capabilities across three cultivatable roles.

**Product Owner (PO)**: accountable for what gets built. Defines product vision, manages user stories, sets priorities. The most important capability is being able to say what will not be built.

**Tech Owner (TO)**: accountable for how it gets built. Leads the agent army through all development. The core work is orchestrating multiple agent workflows, designing module boundaries, and handling exceptions during execution.

**Quality Owner (QO)**: accountable for whether it is built correctly. Independent from development, managing quality across the entire lifecycle, from participation in acceptance criteria definition during the user story stage, through test framework construction, to final end-to-end acceptance.

The QO's independence is the keystone of the entire structure. The TO, who controls the agent army and delivers the output, requires an independent third party to verify it. NASA established an Independent Verification and Validation program after the Challenger disaster. The core principle: software verification must be performed by an organization independent of the developers. The same reasoning applies here.

This team structure **contains no role called "programmer."** The agents write the code. Humans define requirements, set boundaries, review output, and resolve conflicts. Every human function is management work.

A two-person team can also operate: PO doubles as QO and pairs with a TO. Acceptance independence takes a hit, but it is sufficient for smaller projects.

## V. Looking Ahead: The Boundaries Will Keep Moving Up

One clarification: all practices discussed in this article target **large-scale, enterprise-grade AI Coding projects**: long-running databases, cloud collaboration tools, commercial systems requiring sustained maintenance. Small projects do not need this level of process.

The direction that excites us more is different. On open-ended problems, we have found that stepping back and letting agents plan their own paths often produces results beyond expectations. Define only a high-level goal, let the agent draft its approach, build its own tools, and iterate in a loop. The creativity AI demonstrates in this autonomous mode frequently surprises us.

This may signal a trend: as model capability continues to grow, the deterministic boundaries we need to establish will become fewer and broader. Management granularity will shift from micro to macro, from "define every step" to "define the goal, then let go." Today you need detailed execution documents for AI to follow. Perhaps next year only a PRD is needed. The deterministic boundaries will not disappear, but they will keep abstracting upward.

At the same time, the variance in AI's benefit across different people is large. For someone using it optimally, the multiplier may be ten times. For an average user, it may be ten percent.

AI is not a uniform efficiency booster. It is an amplifier of your management capacity and systems thinking.

The craftsmanship engineers have long taken pride in is depreciating. Systems design ability, requirements analysis, deep understanding of the business domain: these have become more valuable than ever.

Agents can build anything you can describe clearly. They cannot decide what is worth building, because they serve as components of the tool rather than users of what they produce.

If building things gives you a sense of purpose, you are living in the right era. Your capacity to manage agents is the ceiling on your creative output.
