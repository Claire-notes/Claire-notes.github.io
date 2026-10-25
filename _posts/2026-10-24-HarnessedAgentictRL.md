---
layout: post
title: Harnessed Agentic RL (01)
---

###### 简单聊聊 Harnessed Agentic RL（简称HAR） 
​
###### Harnessed Agentic RL产生的背景 
背景：Agent = 大语言模型（LLM） + 运行框架（Harness）。
现在的Agent是运行在Harness框架内的模型(LLM/Agent)，不再是一个单独模型(LLM/Agent)与环境交互。Harness负责决策模型如何思考规划、多轮调用工具和行动、感知和管理上下文、多智能体协同、存储和评估结果，Harness已经是Agent的核心组件。

2023-2025：早期的Agent-RL是在训练引擎中实现AgentLoop交互循环(AgentLoop直接写入GPU Cluster)，与环境交互是原生Token-in/Token-out API结构，所以在整个执行过程中每个rollout样本保持着在时间上连续的Token序列流，不存在Decode-encode drift问题。  

2025-至今：Harnessed Agentic RL由于harness在控制工具调用、上下文拼接、流程流转与多智能体协同，一次任务可能会被拆分成动态次数的模型请求，harness管理请求间的状态信息传递和重试，与环境交互主要采用Message API/Response API，随之产生Decode-encode drift问题导致RL训练off-policy。  
同时Harness 越做越复杂——工具链、上下文管理、子agent 调度、失败重试、沙箱隔离，每个harness背后是一套完整的工程系统。想要把整套 harness 逻辑​​重写进训练框架​​，已经不太可能。  
*Decode-encode drift：decode后再encode和原来token序列不同；导致模型训练看到的token序列与此前推理时token序列不同，模型在学习一个非真实的输入（off-policy）*    
因此Harnessed Agentic RL与早期Agent RL有着本质区别，系统设计上也面临不同挑战。  
为什么需要转变到Harnessed Agentic RL：  
1. ​​Harness复杂度​​越来越高，重写入训练引擎成本已经不可接受。
2. 训练-部署一致性​​：在训练harness上学到的策略，在部署harness下可能完全失效。

###### Agentic RL训练范式
训练框架直接持有LLM/Agent运行时状态，需要把Agent的执行过程（rollout）与环境交互循环逻辑写进训练框架(GPU Cluster)，让训练器拿到模型每次生成的（tokenization、轨迹记录、loss mask 构建）。  
本质问题：训练和部署时用的是两套不同的‘harness’。  
🔹 veRL￼
- ​​核心设计​​：原生支持协程式 Agent（白盒），框架直接持有运行时状态。在面对黑盒 Agent 时（外部 CLI 子进程、远程 HTTP 服务），需要走完全不同的集成路径。
- ​​定位​​：白盒是 veRL 的原生模式，黑盒适配需要额外工作。
- 🔹 SLIME v3.0￼
- ​​核心设计​​：白盒路线的另一套实现，训练引擎直接管理 rollout 和环境交互。
- ​​局限​​：和前两者相同。

🔹 aReal

🔹 Molt￼ （NVIDIA NeMo，2026.07）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/molt.jpeg "Polar Framework")

- ​​核心设计​​：PyTorch 原生框架，约 8.6K 行代码，强调紧凑可追溯。单一异步环训练多模态和 MoE 策略。
- ​​定位​​：白盒路线，面向算法研究者频繁改动的需求，强调"整个代码库能装进脑子"。
veRL AgentLoop架构 ：用户实现Agent逻辑，训练和Agent是耦合在一起的，训练需要写一套，推理也需要写一套。

这种训练范式带来三个问题：  
1. ​​训练-部署不一致​​：训练时看到的 prompt 和部署时模型实际看到的 prompt 可能不同。
2. ​​训练框架维护成本高​​：更换不同harness或harness更新 就要重写RL训练逻辑框架(训练代码、适配API格式、处理KVcache、对齐token序列)。
3. 训练数据无法重复利用：原Harness累积的训练数据无法复用，数据资源和GPU资源大量浪费。
除这三个现存问题外，Harnessed agent RL基于这种范式也是行不通的：  
原因1.Harness框架已经庞大复杂到无法写入GPU Clauster（Claud code，codex，Qwen Code，DSH等）。  
原因2.就算Harness能够写入GPUClauster，Decode-encode drif导致的token流断裂和样本动态性会带来RL训练的不稳定或失效。 

​
###### Harnessed Agentic RL训练系统范式
Harnessed Agentic RL​​范式：让部署时的harness直接参与RL-training，训练引擎不持有Harness Agent的运行时状态，只通过API-Proxy记录LLM请求-响应轨迹，使得训练和部署的Harness保持同一。在几乎不修改Agent源码的前提下，可以将不同Harness无缝接入强化学习训练。
基于代理（Proxy-based）的黑盒交互范式已在verl Uni-Agent、AReaL 2.0、slime v0.3.0 逐步采用，虽然整体还在初期仍不完整成熟，但是这种原生实现Harnessed Agentic RL大概率是未来主流范式。
从"训练框架拥有环境交互循环"—>"harness 拥有环境交互循环，训练框架只观察请求-响应对"​​。
Polar 解决了"怎么观察"的问题，但没有处理后续的训练挑战——样本合并、优势计算、损失归一化、后端调度。
​​
###### Pattern1: 🔹 Polar￼ （NVIDIA，2026.05）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/polar.jpeg "Polar Framework")
黑盒路线的开创性工作。
- ​​核心设计​​：在模型 API 端点处放置 Agent Proxy，直接记录推理服务器返回的原始 token IDs 和 logprob，不做 decode-encode。支持两种轨迹构建方式：per_request 和 prefix merging￼ （约 5.4 倍加速）。
- ​​Rollout 架构​​：Rollout Server 负责异步调度，Gateway Node 负责实际执行和工具调用。
- ​​算法​​：使用 GRPO￼ ，不需要 critic 网络。
- ​​效果​​：Qwen3.5-4B + Codex harness，SWE-bench 从 3.8% 提升到 26.4%；Qwen3.5-4B + Claude Code￼ ，从 29.8% 提升到 34.6%。
- ​​局限​​：解决了"怎么观察"的问题，但没有系统处理样本合并、优势计算、损失归一化等后续训练挑战。
1. LLM-API-Gateway
提供兼容 OpenAI 的 LLM 代理接口。Harness 的每次模型调用都会自动关联到对应的 rollout，记录 prompt token IDs、response token IDs 和 nodeprob。Gateway 不参与梯度计算，只做观测和记录。
2. Rollout Controller
负责启动和管理 Agent 执行。支持两种运行方式：​​本地进程​​ 和 ​​Kubernetes 任务​​。原生支持 K8s 意味着不需要依赖 Modal、E2B 等商业沙盒服务，可以直接用自建集群做大规模 rollout。
3. Customized Trainer
负责收集日志、根据前述规则构建训练样本、计算优势值和 loss、执行策略更新。更新时会暂停 API 请求，降低策略陈旧问题。

###### Pattern2: 🔹 Agent LightningV1.0（微软，2026.08）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/agentlightning.jpeg "Polar Framework")

在黑盒路线上做了一次系统性的补全。
- ​​核心设计​​：同样通过 API Gateway 中的 LLM Proxy 拦截请求，但和 Polar 不同，Agent Lightning 在 Gateway 层主动做 retokenization 对齐（统一 chat template），并系统研究了 retokenization shift 的三个来源。
- ​​新增组件​​：Rollout Controller（支持本地进程和 K8s Job，不依赖商业沙盒）+ Customized Trainer（处理样本合并、rollout-level advantage、损失归一化）。
- ​​效果​​：Qwen3.5-9B，仅 6K 样本，SWE-bench Verified 从 41.8% 提升到 56.4%。
- ​​和黑盒的关系​​：Harness 完全不变，训练引擎只观察 request-response 对，是纯黑盒范式。但比 Polar 更系统地处理了后续训练挑战。

###### Pattern3: 🔹 OpenForge￼（微软 / 哥伦比亚，2026.07）
Forge: Scalable Agent RL Framework and Algorithm，trainer、rollout、agent均是分离的结构，其优势就是可以训练黑盒 + 白盒的agent，理论上可以训练任何agent。

- ​​核心设计​​：轻量代理拦截 harness 的 LLM 调用，路由到 RL 推理引擎并录成训练轨迹。K8s 编排器管理沙箱 Pod 生命周期。
- ​​特点​​：接新 harness / 新环境只需要改沙箱，即插即用对接 veRL￼ 。
- ​​定位​​：和 Polar、Agent Lightning 类似，都是黑盒路线，强调"不换 harness 就能训"。


###### Harnessed Agentic RL vs Agentic RL
Harnessed Agentic RL的核心优势是​harness不变、训练和部署一致​​，代价是 token 级信号需要通过各种手段还原，存在 retokenization 等问题。白盒路线的核心优势是 ​训练动态完全可控​​，代价是每换一个 harness 就要重写一遍，且训练-部署不一致，部分Harness较难在训练引擎中实现。
从趋势上看，2026 年的工作明显在向Harnessed Agentic RL倾斜——Polar、Agent Lightning、OpenForge 都是这个方向。背后的逻辑很直接：真实世界的 Agent 都有自己成熟的 harness（Claude Code、Cursor、内部自建的），让研究者把这些 harness 重写一遍既不现实也不合理。


###### Case Study


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


###### Challenges 
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

🔍 Discussion（讨论）
论文最后讨论了 Harnessed Agentic RL 的局限：
- 训练引擎只能看到离散的 LLM 调用，harness 内部状态和工具副作用可能难以完整还原。
- 错误的样本拼接或优势归因会让训练不稳定。
- 跨 harness 迁移仍需更多验证。
- 重试去重、外网隔离和奖励投机的防护还需要持续完善。
- 结果尚未覆盖所有超长 Agent 场景。
整体来说，论文的贡献在于把"用什么 harness 运行"从部署细节提升为训练变量，并提供了一个可复现的测试平台来系统研究这些问题。

###### Citation
Please cite this work as:
Claire. “Harnessed Agentic-RL Optimization”. Claire-notes (Oct 2026). https://claire-notes.github.io/HarnessedAgentictRL

Or use the BibTeX citation:
```json
{
@article{weng2026harness,
  title = {Harnessed Agentic-RL Optimization},
  author = {Claire (qingYe)},
  journal = {claire-notes.github.io},
  year = {2026},
  month = {October},
  url = "https://claire-notes.github.io/HarnessedAgentictRL"
}
```



