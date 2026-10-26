---
layout: post
title: Harnessed Agentic RL (01)
---
###### Harnessed Agentic RL产生的背景 
背景：Agent = 大语言模型（LLM） + 运行框架（Harness）。Agent 的演进与 Harness 的崛起。
现在的Agent是运行在Harness框架内的(LLM/Agent)复合系统，不再是一个单体模型(LLM/Agent)与环境直接交互。Harness负责决策模型如何思考规划、多轮调用工具和行动、感知和管理上下文、多智能体协同及结果存储和评估，Harness已经是Agent的核心组件。

- 2023-2025：早期的Agent-RL，是在训练引擎中直接实现AgentLoop交互循环(直接写入GPU Cluster)，模型与环境交互是原生Token-in/Token-out API结构，所以在整个执行过程中每个rollout样本保持着在时间上连续的Token序列流，天然不存在Decode-encode drift问题，保证了强化学习在完全的 On-policy 状态下高效收敛。  

- 2025-至今：Harnessed Agent，由harness控制工具调用、上下文拼接、流程流转与多智能体协同，一次任务可能会被拆分成动态次数的模型请求，harness与环境交互主要采用Message API/Response API，随之产生Decode-encode drift问题导致RL训练陷入非真实环境off-policy。  
  
- Harness 越来越复杂：工具链、上下文管理、子agent 调度、失败重试、沙箱隔离，每个harness背后是一套完整的控制系统。

> *Decode-encode drift：decode后再encode和原来token序列不同；导致模型训练看到的token序列与此前推理时token序列不同，模型在学习一个非真实的输入（off-policy），破坏算法收敛性*    

为什么转变到Harnessed Agentic RL：  
1. ​​Harness复杂度​​变高：重写入训练引擎已经不可接受，想要把庞大的Harness逻辑在训练框架（如 Megatron-LM 或 FSDP）内部用 PyTorch 重写，边际成本与维护代价极高。
2. 训练-部署一致性（Training-Deployment Consistency）​​：强行在训练框架内使用简化的不同的Harness进行交互训练，再叠加Decode-encode drift问题，会导致在‘训练环境harness’上训练的模型，在‘部署环境harness’下严重失效。

###### Agentic RL训练系统范式
训练框架直接持有LLM/Agent运行时状态，需要把Agent的执行过程（rollout）与环境交互逻辑写进训练框架(GPU Cluster)，训练器拿到模型每次生成的（tokenization、轨迹记录、loss mask） 构建数据。  
本质问题：训练和部署时用的是两套分别实现(可能不同）的‘harness’。  
这类框架主要有：veRL、slime、aReal、Molt。架构范式是同类。  
- ​​核心设计​​：原生框架模式直接持有运行时状态。训练引擎直接管理rollout和环境交互。
- 定位​​：白盒是原生模式，黑盒适配需要额外工作。

🔹 veRL（字节）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/verl.jpeg "veRL Framework")


🔹 Molt （NVIDIA NeMo，2026.07）.  
![这是图片](/images/2026-10-25-HarnessedAgenticRL/molt.jpeg "Molt Framework")

这种范式主要的问题：  
1. ​​训练-部署不一致​​：训练时看到的 prompt 和部署时模型实际看到的 prompt 可能不同。
2. ​​训练维护成本高​​：更换不同harness或harness更新 就要重写RL训练逻辑框架(训练代码、适配API格式、处理KVcache、对齐token序列)。
3. 训练数据低复用率：原Harness累积的训练数据无法复用，数据资源和GPU资源大量浪费。
除这三个现存问题外，Harnessed agent RL基于这种范式也是行不通的：
- 1: Harness框架已经庞大复杂到无法写入GPU Clauster（Claud code，codex，Qwen Code，DSH等）。
- 2：就算Harness经简化后能写入GPUClauster，叠加Decode-encode drif带来的样本动态性加剧会导致RL训练不稳定或失效。 

​
###### Harnessed Agentic RL训练系统范式
Harnessed Agentic RL​​：让部署harness直接参与RL-training。  
基于LLM-API-Proxy，训练引擎不再重写Harness，只通过Proxy记录每次LLM（请求-响应）轨迹，合并组装为样本。使得训练和部署Harness同一，从根本上消除了训练-部署偏差。
这种范式目前主要框架系统有**Polar、AgentLightningV1.0、OpenForge**。
Harnessed Agentic RL范式正从"训练引擎重现交互"转向"训练引擎只做观察"，这不是过渡方案，而是工业界 Agentic RL逐步未来的标准形态。目前verl **Uni-Agent**、AReaL-2.0、slime-v0.3.0 也在落地中。

​
###### Pattern1: 🔹 Polar （NVIDIA，2026.05）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/polar.jpeg "Polar Framework")
Harnessed Agentic RL的开创工作。
- ​​核心设计​​：LLM-endpoint Proxy，记录推理引擎的原始tokenIDs和nodeProb，不做decode-encode。支持两种方式做轨迹重建：per-request和 **prefix-merging （约5.4倍加速）**。
- ​​Rollout架构​​：Rollout Server 负责异步调度，Gateway Node 负责实际执行和工具调用。
- ​​效果1​​：Qwen3.5-4B + Codex，SWE-bench从3.8%提升到26.4%
- 效果2: Qwen3.5-4B + ClaudeCode，SWE-bench从29.8%提升到34.6%
- ​​局限​​：完成解耦架构，样本初步合并方法设计，但在Retokenization、样本合并、优势计算、损失归一等挑战上优化不足。

1. LLM-API-Gateway
提供兼容 OpenAI 的 LLM 代理接口。Harness 的每次模型调用都会自动关联到对应的rollout，记录 prompt token IDs、response token IDs 和 nodeprob。不参与梯度计算，只做观测和记录。
2. Rollout Controller
负责启动和管理 Agent 执行。支持两种运行方式：​​本地进程​​ 和 ​​Kubernetes 任务​​。原生支持 K8s 意味着不需要依赖 Modal、E2B 等商业沙盒服务，可以直接用自建集群做大规模 rollout。
3. Customized Trainer
负责收集日志、根据前述规则构建训练样本、计算优势值和 loss、执行策略更新。更新时会暂停 API 请求，降低策略陈旧问题。
Polar 解决了"怎么观察"的问题，但没有处理后续的训练挑战——样本合并、优势计算、损失归一化、后端调度。

###### Pattern2: 🔹 Agent LightningV1.0（微软，2026.08）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/agentlightning.jpeg "Polar Framework")

在Polar架构解耦路线上，框架更轻量级，训练系统优化更详细。
- ​​核心设计​​：同样通过 API Gateway 中的 LLM Proxy 拦截记录请求，但和 Polar 不同，Agent Lightning 在 Gateway 层主要做了统一Chat-template）；并系统分析了retokenization shift原因和缓解方法。
- ​​Rollout Controller：（支持本地进程和 K8s Job，不依赖商业沙盒）
- Customized Trainer（轨迹合并、rollout-level advantage、损失归一化）。
- ​​效果​​：Qwen3.5-9B，6K 样本，SWE-bench Verified 从 41.8% 提升到 56.4%。

###### Pattern3: 🔹 OpenForgeRL（微软 / 哥伦比亚，2026.07）
![这是图片](/images/2026-10-25-HarnessedAgenticRL/OpenForge.jpeg "OpenForge Framework")
Forge: Scalable Agent RL Framework and Algorithm，trainer、rollout、agent均是分离的结构，其优势就是可以训练黑盒 + 白盒的agent，理论上可以训练任何agent。

- ​​核心设计​​：同样通过API Gateway拦截记录Harness的LLM调用，路由到RL推理引擎并记录轨迹。
- Rollout Orchestrator：启动远程沙箱Harness与环境进行交互并调用LLM，ProxyServer拦截LLM调用、路由至推理引擎、记录轨迹。
- ​​特点​​：接新 harness / 新环境只需要改沙箱，即插即用对接 veRL 。
- ​​定位​​：和 Polar、Agent Lightning 类似，强调"不实现harness就能训"。


###### Harnessed Agentic RL vs Agentic RL
1. 更长程的Personal-Agent/ Work-Agent会更依赖Harnessed-RL优化。传统方法每换Harness就要重写，训推严重不一致。2026年下半年技术趋势开始走向Harnessed Agentic RL，核心是部署时的Harness直接主导交互，实现训练引擎零重写和训推严格一致。

2. API-Proxy架构基本成型：目前止步于“观察记录”，后续训练中：Token级信号还原、样本动态性缓解、长程轨迹合并、优势计算等挑战，现有技术方案还不完善。


###### Case Study，一些实验
![这是图片](/images/2026-10-25-HarnessedAgenticRL/exp.jpeg "Exp Result")


###### Challenges 
1. Retokenization
训练引擎拿到的请求-响应对，需要拼成连续的训练轨迹。但 harness 和训练引擎的 tokenizer、chat template 可能不一致，导致 token 级别的前缀对不上，模型学到的是历史未发生的轨迹。retokenization shift 的三个来源：tokenizer 差异、chat template 差异、inference engine后处理。除chat tempalte问题可解决，其他问题目前没有彻底解决方案。
2. Sample Merging（样本合并）
一个 rollout 可能产生多个请求-响应对，这些对能否合并成一个训练样本，取决于后一个请求的 prompt 是否和前一个请求+响应的 token 级前缀精确一致。实际中约 36% 的 rollout 可以合并成，单次rollout平均产生 2.41个训练样本。
3. Advantage Calculation（优势计算）
如果按单个请求计算优势值，多请求的 rollout 会获得更大的总权重。但由于请求数量是 harness 内部行为的结果，不代表任务更难或更重要，因此优势应该按 ​​完整 rollout 成组计算​​，先在 rollout 内做 token 平均，再对 rollout 平均。
4. Loss Normalization（损失归一化）
类似地，如果直接按样本数量平均 loss，产生更多样本的 rollout 会获得更大的优化权重。损失归一化也需要避免被样本数量扭曲，同样以 rollout 为统计单位。
5. Training Backend Scheduling（训练后端调度）
每个 rollout 产生多少样本、每个样本多长，只有 harness 执行完才能确定，但 GPU 数量和并行配置是固定的。后端需要动态地把可变长度、可变数量的样本映射到固定的训练资源上。
1. 重试去重、外网隔离和奖励投机的防护还需要持续完善。
2. 训练引擎只能看到离散的 LLM 调用，harness 内部状态和工具副作用可能难以完整还原。
3. 错误的样本拼接或优势归因会让训练不稳定。
4. 跨 harness 迁移仍需更多验证。
5. 结果尚未覆盖所有超长 Agent 场景。

###### Citation
Please cite this work as:
Claire. “Harnessed Agentic-RL Optimization”. Claire-notes (Oct 2026). https://claire-notes.github.io/HarnessedAgentictRL

Or use the BibTeX citation:
```json
{
@article{weng2026harness,
  title = {Harnessed Agentic-RL},
  author = {Claire (qingYe)},
  journal = {claire-notes.github.io},
  year = {2026},
  month = {October},
  url = "https://claire-notes.github.io/HarnessedAgentictRL"
}
```
