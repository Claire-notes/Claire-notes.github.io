---
layout: post
title: Harnessed Agentic RL (01)
---

###### 今天聊一聊 Harnessed Agentic RL（简称HAR）  

###### Harnessed Agentic RL产生的背景 
Agent = 大语言模型（LLM） + 运行框架（Harness）。
现在的Agent是运行在Harness框架内的模型(LLM/Agent)，不再是一个单独模型(LLM/Agent)与环境交互。Harness负责决策模型如何思考规划、多轮调用工具和行动、感知和管理上下文、多智能体协同、存储和评估结果，Harness已经是Agent的核心组件。

过去的Agent-RL是在训练引擎中实现AgentLoop交互循环(AgentLoop直接写入GPU Cluster)，与环境交互是原生Token-in/Token-out API结构，所以在整个执行过程中每个rollout样本保持着在时间上连续的Token序列流，不存在Decode-encode drift问题。  

而Harness Agent RL由于harness在控制工具调用、上下文拼接、流程流转与多智能体协同，一次任务可能会被拆分成动态次数的模型请求，harness管理请求间的状态信息传递和重试，与环境交互主要采用Message API/Response API，随之产生Decode-encode drift问题导致RL训练off-policy。
*Decode-encode drift：decode后再encode和原来token序列不同；导致模型训练看到的token序列与此前推理时token序列不同，模型在学习一个非真实的输入（off-policy）*
因此Harnessed Agentic RL与传统Agent RL有着本质区别，算法系统设计上面临不同挑战。  


###### 传统的Agentic RL训练系统范式

把Agent的执行过程（rollout）接入训练是依赖把整个环境交互循环逻辑写进训练框架(GPU Cluster)，让训练器拿到模型每次生成的token序列，这本质是训练和部署是两套分别的‘harness’。这种训练范式带来三个问题：  
1. ​​训练-部署不一致​​：训练时看到的 prompt 和部署时模型实际看到的 prompt 可能不同。
2. ​​训练框架维护成本高​​：更换不同harness或harness更新 就要重写RL训练逻辑框架(训练代码、适配API格式、处理KVcache、对齐token序列)。
3. 训练数据无法重复利用：原Harness累积的训练数据无法复用，数据资源和GPU资源大量浪费。
  
除这三个现存重点问题外，Harness agent基于传统Agent RL训练范式也是行不通的：  
4.Harness框架已经庞大复杂到无法写入GPU Clauster（Claud code，codex，Qwen Code，DSH等）。  
5.就算Harness能够写入GPUClauster，Decode-encode drif导致的token流断裂和样本动态性会带来RL训练的极不稳定甚至失效。 

​​Harnessed Agentic RL​​范式：Training/Execution解耦架构，让部署时的harness直接参与RL-training。  
Harness作为黑盒负责环境交互，同时通过LLM-API-proxy（LLM-Endpoint-Proxy）和推理引擎交互；训练引擎同样LLM-API-Proxy 观察请求-响应轨迹​​。这样harness 不需要写入GPUClauster，使得训练和部署保持一致。在几乎不修改Agent源码的前提下，可以将不同Harness无缝接入强化学习训练。基于代理（Proxy-based）的黑盒交互范式已在verl Uni-Agent、AReaL 2.0、slime v0.3.0 和 Polar 逐步采用，虽然整体还在初期仍不完整成熟，但是这种原生实现Harnessed Agentic RL大概率是未来的主流范式。

###### Harnessed Agentic RL训练系统范式
###### Pattern1: Polar
1. LLM-API-Gateway
提供兼容 OpenAI 的 LLM 代理接口。Harness 的每次模型调用都会自动关联到对应的 rollout，记录 prompt token IDs、response token IDs 和 nodeprob。Gateway 不参与梯度计算，只做观测和记录。
2. Rollout Controller
负责启动和管理 Agent 执行。支持两种运行方式：​​本地进程​​ 和 ​​Kubernetes 任务​​。原生支持 K8s 意味着不需要依赖 Modal、E2B 等商业沙盒服务，可以直接用自建集群做大规模 rollout。
3. Customized Trainer
负责收集日志、根据前述规则构建训练样本、计算优势值和 loss、执行策略更新。更新时会暂停 API 请求，降低策略陈旧问题。

###### Pattern2: AgentlightningV1.0

###### Pattern3: QwenCode？DeepSeekHarness


###### Harnessed Agentic RL Optimization
在三个场景上做了验证：
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


###### Future Challenges 
Challenges（挑战）
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



