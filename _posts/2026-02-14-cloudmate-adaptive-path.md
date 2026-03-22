---
layout: post
title: "End-to-End Evolution for Unknown Failures: CloudMate's Adaptive Path Exploration"
date: 2026-02-14
---

Last month, we hosted Zhao Xiang, the lead of Tencent Cloud's CloudMate intelligent operations system, for an online talk. The discussion centered on a core question: in a continuously changing production environment, how do you build an agent system capable of handling unknown failures?

As large model capabilities improve, integrating AI into development and production environments is inevitable. Operations is a highly dynamic domain: system architectures change, code logic is modified, and microservice topologies shift continuously. A static agent, however capable at deployment, will degrade over time as the gap between its preset knowledge base and the actual environment widens.

Two main engineering approaches address this challenge.

The first is the **bottom-up knowledge engineering** school. This approach relies on fine-grained knowledge management to handle change. Engineers design complex knowledge graphs and synchronize documentation and rules with every release. The fundamental scaling problem: as system components grow, the complexity of knowledge maintenance scales at O(n). Heterogeneous document formats, conflicting content, and retrieval accuracy all become bottlenecks that constrain agent performance.

The second is the **top-down end-to-end evolution** school. This borrows the logic of GPT-series models — intelligence emerging from large-scale data — and abandons manually preset rules in favor of direct learning through interaction with the environment. Just as grammar-rule-based approaches in traditional NLP were displaced by end-to-end training, CloudMate chose this path. Rather than prescribing how to act, the system uses outcome feedback and iterative exploration to discover optimal strategies.

Zhao Xiang argued that achieving this end-to-end self-evolution requires a complete closed loop: **evaluation** sets direction, **mutation** generates candidate paths, and **backtesting** enforces safety. All three are necessary.

> CloudMate runs hundreds of agent instances and handles tens of thousands of failure analysis requests per week. This article walks through the three modules and how they work together to form a continuously evolving system.

## Evaluation: Defining the Boundaries of Capability

Evaluating agent performance in an open operations environment is substantially more difficult than evaluating traditional NLP tasks. Anthropic's 2025 research identified three core challenges: non-deterministic outputs, ambiguous task definitions, and environmental side effects from execution. Inaccurate evaluation causes evolution to drift in the wrong direction.

CloudMate built a dual-track evaluation system that runs objective metrics and subjective reasoning assessment in parallel, with the goal of identifying successful trajectories from exploration and internalizing them.

**Objective metrics** form the baseline layer, focused on execution-level statistics: task completion rate (did the system produce a definitive conclusion within the time limit?), tool call efficiency (were there redundant calls, invalid parameters, or infinite loops?), and end-to-end latency (time elapsed from alert trigger to root cause identification).

**Subjective reasoning assessment** is the core innovation in CloudMate's evaluation framework. In operations diagnosis, a correct conclusion does not imply a correct process — the agent may have guessed the root cause. CloudMate addresses this by using a higher-capability model as a judge to audit the agent's reasoning process. The key audit dimensions are evidence completeness (is each inference the agent makes supported by explicit observational data?), logical coherence (are there causal breaks between reasoning steps?), and intent comprehension (did the agent correctly interpret ambiguous user instructions?).

This dual-track scoring allows the system to precisely identify low-scoring cases. These cases serve as **seeds** for the next stage: they directly trigger the mutation process.

## Mutation: Generating Effective Exploration Trajectories

Once the evaluation module identifies a failure case that needs fixing, CloudMate initiates the mutation process. Because agent exploration runs are long and expensive, random exploration is both costly and ineffective. In CloudMate, mutation is directed search in an unknown solution space. Two complementary path generation strategies are used: **parallel exploration** and **expert guidance**.

### Parallel Exploration: Trading Compute for Intelligence

For most failures with definable states, CloudMate uses large-scale parallel sampling. The system concurrently launches N agent instances in a sandbox environment, expanding the search space by raising the model's temperature parameter or using different backend models.

Consider a "database connection pool exhaustion" scenario. A single agent often falls into a local optimum: check configuration, recommend scaling. In parallel exploration mode, the system generates multiple divergent paths:

- **Path A**: Focuses on resource configuration, recommends increasing `max_connections`. (Evaluated: failed, treats the symptom not the cause)
- **Path B**: Focuses on full-chain tracing, finds high latency on a specific API call. (Evaluated: inconclusive)
- **Path C**: Focuses on internal database state, queries `slow_query_log`, finds a SQL statement missing an index causing connection pile-up. (Evaluated: successful, root cause identified)

This approach — acquiring high-quality trajectories through large-scale sampling — parallels practices in other systems, such as the test-time reasoning strategy in DeepSeekMath-V2 (2025). A reliable evaluation function combined with high-quality exploration trajectories can substantially exceed the base model's reasoning limits and capture diagnostic paths that would rarely be generated under normal conditions.

### Expert Guidance: Behavioral Cloning and Reverse Reasoning

For complex failures that resist random exploration, CloudMate introduces human-machine collaboration. When a human expert intervenes, the system records the complete operation sequence in the background: which monitoring dashboards were checked, which grep commands were run. These high-confidence trajectories are the best training material for the agent.

The system feeds the expert's operation log as a prompt to the agent and asks it to generate an explanation: why did the expert choose to query the slow log at that step? Through this reverse reasoning, the agent converts the expert's implicit intuition into explicit, executable logic chains.

### Knowledge Convergence: Differential Analysis and Rule Extraction

A successful path in isolation is just a single instance. It must be generalized into reusable knowledge. A Critic model compares the original failing path against the new successful path, distilling the key differences into structured knowledge rules that await merging into the main knowledge base.

At this point, an unknown failure case has been converted into a new knowledge patch. However, whether this newly generated rule produces side effects in other scenarios remains unknown. To prevent new knowledge from degrading existing capabilities, the rule must pass sandbox backtesting before it enters the main knowledge base.

## Backtesting: Automated Validation of Knowledge Increments

New knowledge produced by mutation is, before integration, a local optimum: effective only for the current failure. To confirm it generalizes and does not disrupt the system's existing capability structure, it must go through a complete regression testing process.

The backtesting module serves as a quality gate: it prevents the agent from solving new problem A while degrading its ability to handle existing problem B, ensuring every iteration of the knowledge base is a net improvement.

### Full Regression and Capability Anti-Regression

CloudMate built an automated pipeline analogous to continuous integration in software engineering. The core component is a benchmark case library containing a large number of historically resolved typical failure cases.

Each time the mutation module submits a knowledge update request, the system automatically triggers a full regression. The agent instance loads the knowledge base containing the new rule. The agent then re-runs all historical cases in the benchmark library. The system compares pass rates between the old and new versions. If the new rule causes the success rate on any historical case to drop, the update is immediately rejected and returned to the mutation module for correction.

This mechanism ensures cumulative capability growth without regression. It removes the dependency on manual review for evolving the operations system, and guarantees global behavioral stability through deterministic automated testing.

### Engineering Challenge: Environment Decoupling and Snapshot Simulation

Implementing regression testing in the operations domain is harder than in code testing because of data time-sensitivity and environmental dynamism.

A failure case from last year depended on the logs, metrics, and network topology from that moment. The live environment changes continuously. If historical cases are replayed directly against the live monitoring system, the agent receives inconsistent data and generates large volumes of false positives.

To address this, CloudMate built a snapshot-based sandbox simulation architecture. When a benchmark case is recorded, the system captures not only the agent's conversation logic but also the raw return data from all tool calls via a middle-layer protocol. During backtesting, the agent runs in a closed sandbox: all query requests from the agent are intercepted by the system, and the system returns the historical JSON data from the snapshot directly to the agent, bypassing live monitoring entirely.

This approach decouples test execution from the time dimension. The agent reasons inside a frozen historical slice, which ensures reproducibility and objectivity in benchmark testing.

## Building a Deterministic Evolution Loop

CloudMate now operates as a complete self-evolving system. The three modules form an end-to-end engineering loop. The evaluation module defines acceptance criteria and filters candidate solutions, providing stable input for knowledge generation. The mutation module generates candidate solutions through parallel exploration or expert path analysis, continuously expanding the set of viable answers. The backtesting module provides boundary constraints: through full regression in sandbox environments, it rigorously validates candidate knowledge, removes unstable noise, and ensures new rules satisfy the system's global consistency requirements.

## The Engineering Limits of the E2E Approach

Despite the theoretically complete self-evolution loop, CloudMate faces real challenges in deployment.

**Exploration bias: expert guidance still dominates.** Although the system includes a large-scale parallel exploration module, knowledge internalization still depends heavily on expert guidance in current production practice. Parallel exploration is limited by the narrow diversity of its randomness sources, which rely mainly on model temperature or prompt perturbation. For failures with long causal chains and deep logic, pure machine exploration often fails to converge on an optimal solution within a practical compute budget.

**Cold start and benchmark construction cost.** System safety depends on backtesting, and backtesting depends on a high-quality benchmark case library. Zhao Xiang acknowledged in the talk that maintaining the case library is the most expensive part. Each benchmark case requires expert cleaning, labeling, and confirmation. During cold start, this means substantial manual labor.

**Snapshot limitations in the simulation environment.** The current sandbox backtesting mechanism is based primarily on static snapshots of single systems. Failures in modern microservice architectures often involve interactions among multiple components: distributed transaction consistency, cross-service cascading failures. For failures with complex external dependencies, building a high-fidelity isolated sandbox is extremely difficult.

**Long-term convergence of the knowledge base.** As automatically generated rules accumulate, the knowledge base may develop internal logical conflicts, rule redundancy, or overfitting. Whether the system can maintain knowledge base consistency under long-term automated operation remains an open question requiring sustained observation.

## Conclusion: The Boundary Between E2E and Engineering Governance

CloudMate's practice reveals a perspective that has long been underappreciated: the core competitive advantage of an AI agent lies not only in its static capability ceiling at deployment, but in its rate of evolution during operation. Software systems iterate continuously. An agent that cannot co-evolve with the systems it operates will be unsustainable in production.

This end-to-end self-evolution mechanism is a tool for specific problems, not a universal solution for operations. The end-to-end approach attempts to automate away the O(n) complexity of manual knowledge base maintenance, but it does not eliminate complexity itself. It relocates complexity to new dimensions: how to build high-fidelity simulation environments, how to design low-cost evaluation functions, and how to control the entropy growth of automatically generated knowledge.

For static, rule-bound, low-fault-tolerance scenarios (e.g., accounting systems, basic network configuration), traditional deterministic code-based governance remains the irreplaceable foundation. For dynamic, ambiguous, high-dimensional scenarios (e.g., microservice failure localization, performance tuning), self-evolving agents offer a path through the bottleneck.

CloudMate, as an early explorer, simultaneously points toward new paradigms and demonstrates the real challenges of end-to-end approaches in cold start cost, environment dependency, and long-term convergence. It outlines a viable path toward automated knowledge production in dynamic environments without labeled data, while making clear that this path still contains serious engineering problems that deserve careful attention.

The future of operations systems is likely a fusion of human-designed governance and agent autonomy, not a pure version of either. Exploring the ceiling of end-to-end capabilities and the floor of engineering governance will be a long-term challenge for the entire industry.

---

*Compiled from a technical talk by the Tencent Cloud CloudMate intelligent operations system team.*
