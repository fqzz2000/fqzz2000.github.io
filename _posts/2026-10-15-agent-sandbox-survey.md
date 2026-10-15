---
layout: post
title: "A Survey of Agent Sandboxes: Big Tech Is Optimizing the Wrong Target"
date: 2026-10-15
---

As a systems PhD student, one day earlier this year I had a burst of inspiration and got hooked on agent sandboxes. Since then I have read a lot of work on sandbox platforms from both academia and industry. Having finally beaten my procrastination, I wrote this article to lay out the most important phenomena and observations I found along the way. I hope it helps anyone who is also interested in this area.

This article was written the old-fashioned way, by a human; AI was used only for proofreading. It may not always be a pleasure to read, but it should give you a breather in a cyberspace whose air is increasingly made of AI slop. Comments and criticism are very welcome.

**TL;DR**

- Models keep getting cheaper, and sandboxes now account for 30–40% of an agent's total cost.
- Industrial agent sandboxes follow the serverless paradigm, and they spend most of their effort optimizing and advertising "faster creation."
- But creation is not the bottleneck: agents spend most of their time waiting.
- In the training scenario, DeepSeek's DSec solves bursty creation and image distribution, but many other properties remain to be explored.
- Some future directions: use the waiting time, raise deployment density, and rethink state management.

This article targets readers with a systems background; some familiarity with virtualization helps. Readers without it are advised to read under the supervision of an adult LLM.

## Introduction: Agent Infrastructure Is Becoming the New Cost Bottleneck

Over the past two years, large language models have gone from powering chatbots to driving agents that complete complex tasks: the model can call tools, run commands, and read and write files. Coding agents such as Claude Code and Codex have become everyday development tools for many people. Products like Manus, Lovable, and Replit run agents in the cloud to do development work for their users, and personal agents like Meta Muse send and receive email and make phone calls for you. For safety, these agents' execution environments usually live in sandboxes. Each agent gets an isolated environment, and when an agent makes a mistake or is attacked, the damage does not reach the host or other users.

The conventional wisdom was that, compared with expensive model inference, the cost of sandboxes is negligible, so most research focused on security. But today model prices are falling fast. Flagship models still charge tens of dollars per million output tokens, while budget models like DeepSeek V4-Flash cost only a few percent of that and can already drive agents through complex tasks reliably. As one cost shrinks, sandboxes become a part of agent execution cost that can no longer be ignored. TrEnv-X [1] estimates, using AWS Lambda pricing, that the execution environment can account for up to 42% of the total cost for some agents. Our lab reached a similar conclusion: we collected 256 coding-agent trajectories (Codex driving DeepSeek V4-Flash, with tasks from SWE-bench and Multi-SWE-bench) and replayed them, and the sandbox's median share of the total cost was 32%.

Agent sandboxes have become a new cost bottleneck, and they deserve a careful look.

## Where Agent Sandboxes Are Used

Before walking through the agent sandbox work in academia and industry, some background is needed. Agent sandboxes actually cover two very different scenarios, and it is important to keep them apart.

The first scenario, which I call the **serving scenario**, is infrastructure for agent products like Manus and Replit. Vendors deploy large numbers of sandboxes in their clusters as the execution environments in which users direct LLMs to complete their tasks. Most industrial products and open-source projects, such as E2B [2], Daytona [3], Tencent's CubeSandbox [4], and Alibaba's OpenSandbox [5], focus on this scenario.

The second scenario is helping LLMs train, which I call the **training scenario**. An important step in LLM training is post-training. The basic idea is to throw the model into a "training ground" and let it hone its skills in real environments. For example, to train a model's coding ability, post-training assigns it a large number of coding tasks, lets it complete them in sandboxes, scores the results, and updates the model's parameters accordingly. This scenario has some unique requirements compared with serving. Serving faces many independent sessions that each arrive and finish on their own; training launches large numbers of sandboxes in batches at each training step. The same task is usually sampled into multiple trajectories, and these sandboxes start from the same environment and behave similarly. Scheduling must also be coordinated with the GPU cluster to use training compute as efficiently as possible. Large-scale training sandbox infrastructure is mostly built in-house by model vendors, and little of it is public; most public work comes from universities or industry–academia collaborations. Because the field is so young, most results are still preprints, but they offer plenty of interesting inspiration. One especially valuable reference is DSec [6], a collaboration between DeepSeek and Tsinghua University that describes DeepSeek's internal agent sandbox infrastructure.

In the rest of this article, I introduce existing work in each of the two scenarios, distill the important problems and challenges they share, and hope to offer some inspiration for the development of agent sandboxes.

## Agent Sandboxes in the Serving Scenario

The serving scenario is currently the "liveliest" part of the field. Hot startups like Daytona [3] and E2B [2] keep popping up, and Google [7], AWS [8], Tencent [4], and Alibaba [5] have all launched their own agent sandboxes. These companies also loudly share how well their sandboxes perform on their blogs and social media accounts, painting a picture of a field bursting with vitality.

However, if you look closely at their implementations, you will find them remarkably uniform. They follow the traditional serverless paradigm and differ mainly in generic optimizations.

Take Tencent's CubeSandbox [4] as an example. Its request path is CubeAPI → CubeMaster → Cubelet → CubeShim → MicroVM. CubeAPI exposes an E2B-compatible interface; CubeMaster is a centralized scheduler that decides which node a sandbox goes to; Cubelet is the node agent; CubeShim implements containerd's shim interface; and the hypervisor at the bottom is a fork of Cloud Hypervisor. This maps almost one-to-one onto the Kubernetes layering of API Server → scheduler → kubelet → container runtime. When creating a sandbox, CubeSandbox does not actually boot a VM but restores one from a template: the template is built from an OCI image, cold-booted once, and saved as a memory snapshot; new sandboxes map the same snapshot file, and disks are copied with XFS reflink. This follows the same idea as Firecracker snapshot restore and AWS Lambda SnapStart. As for lifecycle, each user session gets one sandbox; a background process scans periodically and suspends a sandbox only after an idle timeout, resuming it on the next request.

Alibaba's OpenSandbox [5] is built directly on Docker and Kubernetes. Its default backend is Docker. On Kubernetes, it uses a BatchSandbox resource to maintain a pool of pre-started Pods as a warm pool, and switches to gVisor or Kata via RuntimeClass when stronger isolation is needed. To bypass the latency of the Kubernetes control plane, it also offers a Fast Sandbox backend that runs Firecracker VMs inside pre-started Pods and restores them from snapshots.

These two examples show that this field bursting with vitality is, in paradigm, a souped-up version of serverless. The scheduling side is still declaring sandbox objects, scoring nodes, overcommitting resources, and pre-warming instances; the execution side is still traditional sandboxes like Firecracker, gVisor, and Docker. Every vendor is racing ahead inside the serverless comfort zone, pouring engineering effort into "creating sandboxes faster": snapshot restore, warm pools, bypassing the Kubernetes control plane. The usual suspects.

But an agent session does not spend most of its time on creation. An analysis of production traces shows that the median coding-agent session spends only about 20% of its time executing [12]. Moreover, these platforms allocate one sandbox per agent session, so a sandbox lives as long as its agent. These sandboxes routinely live for minutes or even tens of minutes, which makes the startup speed of a single sandbox nearly irrelevant.

(P.S. To be fair, these implementations do contain real engineering contributions. For example, CubeSandbox uses virtio-pmem + DAX to save memory and implements a fine-grained agent permission system with eBPF.)

## Agent Sandboxes in the Training Scenario

The training scenario is much quieter. Most related work comes from universities, and much of it is work in progress with rough edges. But the beloved DeepSeek, together with Tsinghua University, very generously shared the architecture of DSec, their internal agent sandbox platform, along with many measurements from their production environment. In this section, I mainly use the DSec paper as material to introduce the challenges and opportunities agent sandboxes face.

DSec is DeepSeek's internal agent sandbox platform. Strictly speaking, it is not designed only for LLM post-training; it is general infrastructure that handles both regular inference tasks and training tasks. According to the paper, DSec is deployed on multiple production clusters inside DeepSeek, each with about 160 nodes, 30,000 cores, and 250 TB of physical memory. Each cluster handles 3 million sandboxes per day, with peak concurrency reaching 380,000 sandboxes and 5,000 sandboxes created per second [6].

At this scale, DSec faces a very tough test. Each node runs about 2,400 sandboxes concurrently on average, about 20 times the per-node limit recommended by the Kubernetes documentation (110 Pods) and more than 10 times E2B's default per-node limit (200). Several of their production observations are especially interesting.

1. **Sandbox creation during training comes in bursts.** A single training or evaluation job may need to release tens of thousands of sandboxes at once. The figure below shows the distribution of the number of sandboxes created per task at DeepSeek in early 2026 (DSec Fig. 2). In DeepSeek's production environment, the median number of sandboxes for a container task is 2,528, and the p99 reaches 16,388.

   ![Sandbox count per task (DSec Fig. 2)](/assets/img/agent-sandbox/dsec-fig2.png)

2. **Environments are extremely diverse.** Because training and evaluation involve a huge number of different images, image pulling itself becomes a major bottleneck. The table below shows the environment images used in DSec's production environment over one week (DSec Tab. 2). Containers alone used 11,000 base images and 100,000 workspaces, 82.8 TB in total. MicroVMs used few base images, but still 53,000 different workspace images.

   ![Environment artifacts active during one production week (DSec Tab. 2)](/assets/img/agent-sandbox/dsec-tab2.png)

3. **Sandboxes are long-lived and stateful, but use resources sparsely.** A sandbox involves many interactions with the LLM and has to wait for inference, so the CPU is idle most of the time; yet sandboxes live a long time (median about 17 minutes for containers, p99 over 3 hours), holding memory throughout. The figure below shows how a typical sandbox's CPU and memory change over time (DSec Fig. 3). After setup, the CPU shows only sporadic spikes, while memory stays occupied and slowly grows.

   ![A representative sandbox execution (DSec Fig. 3)](/assets/img/agent-sandbox/dsec-fig3.png)

DSec's approach to problems 1 and 2 is simple. Since the main bottleneck of sandbox creation is image pulling, DSec built its own image management system on top of 3FS, their distributed file system. The core ideas are "layering" and "on-demand pulling." An image may be huge, but logically it consists of three independent parts: (1) the base operating system environment, (2) the workspace containing the target code and other things the agent needs for its task, and (3) the agent harness itself (called a toolkit in the paper). And because they found that an agent usually accesses only a small fraction of an image's data while running a task, they adopted a file system called EROFS [9], which supports reading and decompressing compressed data on demand. When images are built, each layer (base environment, workspace, toolkit) is made an independent layer, and the container runtime assembles them on the spot with overlayfs when a container starts. This greatly reduces the number of images the image service has to manage. When a container is created, 3FS does not send the complete image to the node; it only transfers the metadata, and pulls the actual content on demand when it is accessed. For microVMs, they adopted a similar design.

With this approach, in a post-training RL experiment that launched 8,192 containers in parallel, their pull speed nearly matched having all images available locally, 1.71 times faster than the default Docker pull, while disk writes dropped by 57%.

![On-demand EROFS pulling vs. eager Docker pulling (DSec Fig. 10)](/assets/img/agent-sandbox/dsec-fig10.png)

(P.S. If you have read TrEnv, another paper by DSec's first author, you will notice that the ideas of separating metadata from actual data and of layering to speed up startup come straight from TrEnv; DSec just extends the CXL memory pool to DeepSeek's 3FS, and switches the layered, reused components from namespaces and cgroups to the different parts of an image.)

(P.P.S. If you haven't read TrEnv, I strongly recommend it. It is one of the best systems papers I have ever read: [https://dl.acm.org/doi/10.1145/3694715.3695967](https://dl.acm.org/doi/10.1145/3694715.3695967))

For problem 3, DSec takes an approach similar to CubeSandbox's. Since the main memory pressure comes from page cache duplicated inside VMs, they use virtio-pmem + DAX to let multiple microVMs share a single copy of the read-only layers' page cache on the host.

So DSec solves bursty creation and environment diversity well. But for the "long-lived, stateful, resource-sparse" behavior revealed by observation 3, it still relies on off-the-shelf techniques like virtio-pmem and memory reclamation: sandboxes are still allocated per session and stay alive while waiting.

The DSec paper contains many more interesting experiments, designs, and observations, but for space reasons I won't cover its full implementation here, only the parts I think are most relevant. Interested readers can refer to DeepSeek's official post ([in Chinese](https://zhuanlan.zhihu.com/p/2088265189233779592)) and the original paper [6].

## The Future of Agent Sandboxes

By now you have probably noticed a number of subtle mismatches. In both the serving and the training scenario, existing solutions are fairly plain. Plain is not a bad thing, of course, but as systems researchers we want to find clear tradeoffs and boundaries for the problems DeepSeek observed in production. Today, however, serving-side agent sandboxes merely add some simple engineering optimizations on top of serverless, and although DSec offers some direct and effective solutions to its observations, the design space is far from fully explored.

The problems raised in the DSec paper are indeed general. Similar observations have been made and confirmed both in research I took part in [10] and in other papers [11, 12].

So in this section, I list a few problems that I personally consider valuable from a systems research perspective, and I hope they spark some thinking and discussion.

1. **How to make good use of inference time.** Back to the question left open earlier. In both scenarios, existing platforms optimize sandbox creation, while sandboxes actually spend most of their time waiting. I think the least explored question is how to make effective use of LLM inference time. An agent's execution alternates between LLM inference and tool execution. Waiting for the model's output takes anywhere from seconds to hundreds of seconds, which is an absurdly long time by systems standards. The VM cold start that serverless has complained about for years is lightning fast on this time scale. We have found that simply pausing or deleting the sandbox while the agent waits for inference, and bringing it back when the next tool call arrives, already improves throughput substantially. AgentSysBench [12] observed something similar: offloading sandboxes during model inference reduced average memory by 4.6× while increasing end-to-end latency by less than 0.5%. What else could this time be used for, and how should sandbox work be scheduled to use it better? Is a per-session sandbox even the optimal design, or should sandboxes be managed at a different granularity? Today everyone uses snapshots to preserve memory state, but do we really need to snapshot all of it?

2. **How to increase deployment density.** DSec and many other reports note [6, 10, 11, 12] that, because of model inference and the variety of tools invoked, the average CPU and memory utilization of each sandbox is extremely low. DSec also notes that heavy overcommit is needed to use resources effectively. However, many overcommit strategies from traditional cloud computing don't transfer well to agent sandboxes. For example, serverless has a lot of work on predicting a function's resource consumption from historical data, but an agent's next action is hard to predict, and different actions consume very different resources. How do we overcommit as much as possible without hurting sandbox stability through contention or OOM? There is a large design space to explore here.

3. **Local state vs. remote state.** Another problem agent sandboxes face is how to handle state. Traditional cloud workloads love being stateless, which gave rise to architectures like the separation of compute and storage. Agent sandboxes, however, are exactly the kind of long-running, stateful workload that cloud computing hates most: modified files, the shell environment, and background services all have to be kept until the session ends. How to manage this state efficiently is still wide open. The problem is even more extreme in training: when a GPU training job is preempted or synchronizes weights, tens of thousands of sandboxes pause at the same time. Where should their state go, how many resources should it occupy during the pause, and can they be rescheduled to other nodes when they resume? None of these have good answers yet.

Beyond these, several preprints, though less complete, explore some interesting directions:

1. AgentZip [13] compresses memory into a dedicated memory pool while the agent waits for inference and decompresses it on demand, to save memory.
2. DeltaBox [14] proposes a hot-pluggable overlayfs + CRIU design for fast checkpoint/restore of agent state.
3. SpecBox [15] proposes a new agent architecture: instead of giving each agent its own sandbox, it ties sandbox lifecycles to tools and tries to "predict" which tool the agent will call via semantic search, to hide cold-start time.
4. Tide [16] performs fine-grained memory management during an agent's run, using time spent waiting for model output to reduce the system's memory consumption.

The rise of coding agents is also giving some half-forgotten isolation techniques a new life. Take libOSes: HyperFlux [17] allocates CPU cores across libOS VMs at microsecond speed, and this kind of capability might spark something interesting with workloads like agents, whose resource consumption is hard to predict.

## Conclusion

As this article shows, although many companies and researchers have invested in agent sandboxes, the field as a whole is still at an early stage. Many characteristics of agents have not been fully exploited or analyzed. I firmly believe that agent workloads may reshape, to a large extent, the architecture of cloud systems and the abstractions they need to provide. There are many interesting and valuable systems problems waiting to be explored. I hope this article helps interested readers see the big picture of the field, so that we can push it forward together.

Discussion and criticism are very welcome!

## References

[1] Jialiang Huang, Teng Ma, Zheng Liu, et al. TrEnv-X: Transparently Share Serverless Execution Environments Across Different Functions and Nodes. ACM Transactions on Computer Systems (TOCS), 2026.

[2] E2B. [https://github.com/e2b-dev/E2B](https://github.com/e2b-dev/E2B); infrastructure: [https://github.com/e2b-dev/infra](https://github.com/e2b-dev/infra)

[3] Daytona. [https://github.com/daytonaio/daytona](https://github.com/daytonaio/daytona) (closed-source since June 2026; last public version v0.190.0)

[4] Tencent Cloud. CubeSandbox. [https://github.com/TencentCloud/CubeSandbox](https://github.com/TencentCloud/CubeSandbox)

[5] Alibaba. OpenSandbox. [https://github.com/alibaba/OpenSandbox](https://github.com/alibaba/OpenSandbox)

[6] Jialiang Huang, et al. DeepSeek Elastic Compute (DSec): A Sandbox Infrastructure for Effective Agentic Training at Scale. arXiv:2609.22978, 2026. (DeepSeek-AI & Tsinghua University)

[7] Google. Agent Sandbox (kubernetes-sigs/agent-sandbox). [https://github.com/kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox); GKE Agent Sandbox: [https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox)

[8] Amazon Web Services. Amazon Bedrock AgentCore Runtime. [https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html)

[9] Xiang Gao, et al. EROFS: A Compression-friendly Readonly File System for Resource-scarce Devices. USENIX ATC 2019.

[10] Yusheng Zheng, Jiakun Fan, Quanzhi Fu, Yiwei Yang, Wei Zhang, Andi Quinn. AgentCgroup: Understanding and Controlling OS Resources of AI Agents. arXiv:2602.09345, 2026.

[11] Leonid Kondrashov, Hongrui Liu, JooYoung Park, et al. Rethinking AI Cloud Infrastructure for Agentic Serving Systems with the Aries Experimentation Framework. arXiv:2607.29069, 2026.

[12] Chaokun Chang, Yukun Zhou, Kaihua Fu, et al. From LLM Inference to Agentic Workloads: Characterization and Implications for Serving Systems. arXiv:2608.15127, 2026.

[13] Mengming Li, Ceyu Xu, Qijun Zhang, Jiangnan Yu, Xiangfeng Sun, Haohui Mai, Zhiyao Xie. Memory Compression for High-Fanout Agent Sandboxes. arXiv:2609.11294, 2026.

[14] Yunpeng Dong, Jingkai He, Shiqi Liu, et al. DeltaBox: Scaling Stateful AI Agents with Millisecond-Level Sandbox Checkpoint/Rollback. arXiv:2605.22781, 2026.

[15] Yihui Zhang, Tianyu Wo, Jinghao Wang, et al. SpecBox: Speculative Sandbox Scheduling for Efficient LLM Agent Serving. arXiv:2607.23933, 2026.

[16] Yiyang Wu, Chengfan Liao, Jinyu Gu. Tide: Reclaiming Phased Memory in Agent MicroVMs. arXiv:2609.40082, 2026.

[17] Yibo Yan, Seo Jin Park. Offering Microsecond-Scale Cross-VM Core Elasticity on Colocated Lightweight Virtual Machines (HyperFlux). arXiv:2608.12633, 2026.
