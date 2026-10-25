---
layout: post
title: Harnessed Agentic RL (01)
---

###### 今天聊一聊 Harnessed Agentic RL  
Agent = LLM + Harness.  
Modern agents do not operate as standalone LLMs. They run inside **Harness** that manage long-running context, multi-turn tool use, context, and multi-agent orchestration which makes the harness a critical component. How to **directly porting the deploy-time Harness into model RL-training**, thereby narrowing the gap between training and actual use is the term **Harnessed Agentic RL**.

The **Harnessed Agentic RL** differs fundamentally from traditional agentic RL and introduces a new set of challenges.
In traditional agentic RL, the training engine owns the environment interaction loop. The agent loop is embedded directly within the GPU cluster, presenting a native token-in/token-out API structure，maintaining a continuous token sequence stream throughout the execution.
In harnessed agentic RL, 
1.Harness框架已经无法写入GPU Clauster：像Claud code，codex，Qwen Code这样的Harness已经复杂到无法放入GPU Cluster.
2.更换Harness框架或Harness更新：RL框架需要跟随重构，维护成本高. 不同的Harness框架，各有不同的工具调用方式、多轮对话管理、上下文处理逻辑。想要做RL训练，必须把Agent的执行过程（rollout）接入训练流水线，让训练器拿到模型每次生成的token 序列。每换一个Harness框架，这套接入工程就得重做一遍，改训练代码、适配 API 格式、处理 KV cache、对齐 token 序列……工作量巨大.
3.不同Harness的RL训练数据怎样复用

the harness owns this loop, while the training engine observes a sequence of LLM request-response pairs.  How to model and assemble these calls into training samples remains an open question.

基于此，training/execution解耦架构。通过大模型 API 端点代理（LLM Endpoint Proxy），在几乎不修改Agent源码的前提下，将Harness无缝接入强化学习训练。It connects arbitrary agents to RL training through an LLM endpoint, with almost no changes to the agent. 
如今，这种基于代理（Proxy-based）的黑盒交互范式已成为行业标准，并被 verl Uni-Agent、AReaL 2.0、slime v0.3.0 和 Polar 等主流框架广泛采用，原生地实现了受框架约束的智能体强化学习（Harnessed Agentic RL）。
such proxy-based approach has become more common in
frameworks such as verl Uni-Agent [12], AReaL 2.0 [13], slime v0.3.0 [10], and Polar [14], which
naturally enables enables RL training with agent harnesses.

参考小红书与全网15篇内容

现在信息比较充分了，我来按论文原始内容的顺序，逐部分梳理这篇论文。
📄 Abstract（摘要）
论文开篇点明核心问题：现代 Agent 不是孤立的大模型，而是运行在 ​​harness​​ 中的——harness 管理工具、上下文和控制流。传统的 Agent RL 框架（OpenAgents、APOLLO 2.0、SLIME v3.0）都需要重写 harness，而 ​​Polar​​ 虽然引入了 proxy-based 训练，但整体仍不完整。
这篇论文提出 ​​Harnessed Agentic RL​​ 这一新范式：让部署时的 harness 直接参与模型 post-training，从而缩小训练和实际使用之间的差距。
作者识别出 Harnessed Agentic RL 的几个核心挑战：​​retokenization、样本合并、优势计算、损失归一化、测试集调度​​。如果处理不当，会导致训练无效或不稳定。
论文同时开源了 ​​Agent Lightning v1.0​​ 框架，约 3500 行代码，在 coding agent 实验中仅用 6K 训练样本，将 Qwen3.5-9B 在 SWE-bench Verified￼ 上的 Pass@1 从 41.8% 提升到 56.4%。
🔬 Introduction（引言）
问题动机
真实世界的 Agent 由 harness 控制工具调用、上下文拼接和流程控制。一次任务会拆分成数量不定的模型请求，harness 负责管理这些请求之间的状态传递和重试逻辑。
传统 Agentic RL 的做法是把整个环境交互循环搬进训练框架，训练时和部署时用的是两套不同的 harness。这导致两个问题：
1. ​​训练-部署不一致​​：训练时看到的 prompt 和部署时模型实际看到的 prompt 可能不同。
2. ​​工程成本高​​：每换一个 harness 就要重写一遍训练逻辑。
核心主张
论文提出 Harnessed Agentic RL：​​harness 继续负责环境交互，训练引擎只通过一个 LLM Proxy 观察请求-响应轨迹​​。这样 harness 不需要重写，训练和部署保持一致。
但这也带来了一组新的挑战，论文逐一分析并给出了解决方案。
⚙️ Challenges（挑战）
1. Retokenization
训练引擎拿到的请求-响应对，如果要拼成连续的训练轨迹，需要把文本 decode 再 re-encode 成 token IDs。但 harness 和训练引擎的 tokenizer、chat template 可能不一致，导致 token 级别的前缀对不上，模型学到的是它从未见过的 prompt。
论文分析了 retokenization shift 的三个来源：tokenizer 差异、chat template 差异、inference engine 的后处理。其中 ​​只有 chat template 不一致是可以解决的​​（通过统一 template），其他两种 shift 无法完全消除。
2. Sample Merging（样本合并）
一个 rollout 可能产生多个请求-响应对，这些对能否合并成一个训练样本，取决于后一个请求的 prompt 是否和前一个请求+响应的 token 级前缀精确一致。论文发现只有约 36% 的 rollout 可以合并成一行，平均产生 2.41 个训练样本。
3. Advantage Calculation（优势计算）
如果按单个请求计算优势值，多请求的 rollout 会获得更大的总权重。但由于请求数量是 harness 内部行为的结果，不代表任务更难或更重要，因此优势应该按 ​​完整 rollout 成组计算​​，先在 rollout 内做 token 平均，再对 rollout 平均。
4. Loss Normalization（损失归一化）
类似地，如果直接按样本数量平均 loss，产生更多样本的 rollout 会获得更大的优化权重。损失归一化也需要避免被样本数量扭曲，同样以 rollout 为统计单位。
5. Training Backend Scheduling（训练后端调度）
每个 rollout 产生多少样本、每个样本多长，只有 harness 执行完才能确定，但 GPU 数量和并行配置是固定的。后端需要动态地把可变长度、可变数量的样本映射到固定的训练资源上。
🏗️ System Design（系统设计）
Agent Lightning v1.0 由三个核心组件构成：
1. API Gateway
提供兼容 OpenAI 的 LLM 代理接口。Harness 的每次模型调用都会自动关联到对应的 rollout，记录 prompt token IDs、response token IDs 和 logprob。Gateway 不参与梯度计算，只做观测和记录。
2. Rollout Controller
负责启动和管理 Agent 执行。支持两种运行方式：​​本地进程​​ 和 ​​Kubernetes 任务​​。原生支持 K8s 意味着不需要依赖 Modal、E2B 等商业沙盒服务，可以直接用自建集群做大规模 rollout。
3. Customized Trainer
负责收集日志、根据前述规则构建训练样本、计算优势值和 loss、执行策略更新。更新时会暂停 API 请求，降低策略陈旧问题。
整个框架约 3500 行代码，强调简单性作为第一设计原则。
📊 Experiments（实验）
论文在三个场景上做了验证：
1. 通用指令遵循 Agent
在 HotpotQA 上，Llama-3.2-3B 的验证奖励从 25.1 提升到 41.7。
2. 搜索 Agent
在 Qwen3-4B 上，指令任务从 51.9 提升到 70.2。
3. Coding Agent（重点实验）
基于 Qwen3.5-9B，约 6K 训练样本，400 条测试隔离了 Git 和外网以防奖励投机。关键消融实验对比了三种设置：
- rollout-level advantage + 归一化：第 128 步达到 38.2
- 样本级 advantage：35.0
- 只改 advantage 不归一化：33.1
最终在 SWE-bench Verified 上，第 208 步 Pass@1 从 41.8% 提升到 ​​56.4%​​，绝对提升 14.6 个百分点。
🔍 Discussion（讨论）
论文最后讨论了 Harnessed Agentic RL 的局限：
- 训练引擎只能看到离散的 LLM 调用，harness 内部状态和工具副作用可能难以完整还原。
- 错误的样本拼接或优势归因会让训练不稳定。
- 跨 harness 迁移仍需更多验证。
- 重试去重、外网隔离和奖励投机的防护还需要持续完善。
- 结果尚未覆盖所有超长 Agent 场景。
整体来说，论文的贡献在于把"用什么 harness 运行"从部署细节提升为训练变量，并提供了一个可复现的测试平台来系统研究这些问题。

###### Agentic-RL框架 VS Harnessed-Agentic-RL框架
在真实长程任务中，Agent 仍然容易出错，尤其是在超长程，多轮交互、或不熟悉的工具场景下。Agent 已经够强，但还不够可靠，需要 RL-training 来进一步提升。
Agent 在执行过程中会自然产生大量真实的交互数据（轨迹），这些数据比人工标注的数据更丰富、更贴近真实使用场景。而 RL 的优势在于​​基于结果奖励、不需要每一步都标注​​，天然适合 Agent 这种多步决策的场景。

​​第一类：传统 Agentic RL 框架​​（OpenAgents、APOLLO 2.0、SLIME v3.0 等）
- 核心做法：把 Agent 的完整环境交互循环搬进训练框架，训练引擎直接管理 rollout。
- 核心问题：要求开发者把 Agent ​​重新实现到训练框架中​​，导致训练时和部署时用的是两套不同的 harness。
- 后果：训练-部署不一致，且每换一个 harness 就要重写一遍训练逻辑，工程成本极高。
​​
第二类：API-Proxy-based 方案​​
- 核心做法：在 API 边界处放一个 proxy 来拦截请求、记录原始 token，绕开重写 harness 的问题。
- 论文的评价：API-Proxy向前迈了一步，但​​整体仍不完整​​——它解决了"怎么观察"的问题，但没有系统处理后续的训练挑战（样本合并、优势计算、损失归一化等）。
这一步的目的是​​划清战场​​：现有方案要么完全重写 harness（成本高、不一致），要么只做了一部分（不完整）。都需要一个更系统的方案。

第三步：提出 Harnessed Agentic RL 范式
​​Harnessed Agentic RL​​ = 让部署时的 harness 直接参与模型的 post-training，训练引擎只通过一个 LLM Proxy 观察请求-响应轨迹。
这里有一个关键的范式翻转：
- 传统 Agentic RL：​​训练引擎拥有环境交互循环​​。
- Harnessed Agentic RL：​​Harness 拥有环境交互循环​​，训练引擎只能观察到一系列 request-response 对。
论文强调，这种翻转带来的好处是：harness 不需要重写，训练和部署保持一致。

第四步：但这种范式翻转带来了新的挑战
Harnessed Agentic RL 不是简单地把 proxy 插上就能用，它有全新的挑战：
1. ​​Retokenization​​：训练引擎拿到的文本要重新编码成 token，但 harness 和训练引擎的 tokenizer、chat template 可能不一致。
2. ​​Sample Merging​​：一个 rollout 产生多个 request-response 对，哪些能合并成一个训练样本、哪些不能，取决于 token 级前缀是否精确一致。
3. ​​Advantage Calculation​​：按单个请求算优势会导致多请求的 rollout 获得更大权重，需要按完整 rollout 成组计算。
4. ​​Loss Normalization​​：类似地，损失归一化也需要以 rollout 为统计单位，避免被样本数量扭曲。
5. ​​Training Backend Scheduling​​：每个 rollout 产生多少样本、每个样本多长是不确定的，但 GPU 资源是固定的，需要动态调度。
​​
1. Agent 很强但不完美 → 需要 post-training
2. RL 比 SFT 更适合 Agent 场景 → 确定技术路线
3. 现有 RL 方案要么重写 harness 要么不完整 → 指出空白
4. 提出 Harnessed Agentic RL 新范式 → 填补空白
5. 但这种范式带来一组新挑战 → 说明问题的复杂性
6. 开源 Agent Lightning v1.0 来系统研究和验证 → 给出解决方案
整个 Introduction 的逻辑非常紧凑，每一步都在为下一步做铺垫，没有多余的背景堆砌。


###### 一些挑战 


###### 一些实验结果比对


