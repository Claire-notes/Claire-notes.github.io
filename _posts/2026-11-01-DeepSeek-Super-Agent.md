---
layout: post
title: "Agent 架构 (01)：DeepSeek Super-agent 的多任务设计"
date: 2026-11-01 01:00:00 +0800
excerpt: "一个人的注意力只能同时跟进三五个 Session，这就是多任务的天花板。DeepSeek 的解法是把 Agent 从会话里解绑：Router 调度、Worker 执行、事件驱动、上下文分层落盘。"
---

## 问题：多任务的上限卡在了人的注意力
DeepSeek：Super-agent 设计探索，迈向更自主的 Agent 系统

现有 Agent 以单个 Session 为基本单位，不同 Session 彼此独立。用户并行多个任务时，要手动切换会话、手动管理上下文。DeepSeek 团队观察到的数字是：**一个人通常只能同时跟进 3 到 5 个活跃 Session。**

这句话点出了问题的本质：**单任务的上限取决于模型能力，多任务的上限取决于人的注意力。**

这个瓶颈在 Coding Agent 场景尤其明显。一个 PR 可能要改好几轮，一个 Bug 可能在群里讨论半天才定位，一个需求可能跨 IM、邮件、代码评审三个渠道推进。但每个渠道都是独立 Session，上下文互不相通，用户只能自己在脑子里拼合信息，还要记住每个任务进行到哪一步。

常见的「增强」方案——CLI、MCP、外部连接器——能让 Agent 接入更多外部消息，但解决不了三个根本问题：

1. **同步阻塞**：一条消息触发一个 Session，串行执行完之前，新消息要么被忽略，要么另开一个 Session，上下文越来越散。
2. **多任务上下文混乱**：不同任务的中间状态混在同一个 Session 里，模型容易遗忘、容易幻觉。
3. **本地离线即停止**：Agent 跑在本地设备上，关机就停了，没有持久化的执行状态。

所以 Super-agent 要做的不是「让 Session 更多」，而是**把 Agent 从单次手动会话中解绑，变成一个跨会话保持上下文、同时推进多项任务且互不干扰的连续实体。**

![Session 中心与 Goal 中心](/images/2026-11-01-DeepSeek-Super-Agent/fig1-session-vs-goal.webp "两种组织方式")

*左边：每个渠道一个会话，上下文靠人脑缝合；右边：一个 Agent 下挂多个任务，上下文跟着任务走。*

## 方法：Router + Worker 双层架构
Super-agent 的设计目标可以概括为四个词：

- **连续**：在 IM、GitHub 评论、邮件等不同界面交互，感觉始终只有一个 Agent
- **高效**：响应快，能并行处理多个任务，不必等上一条完成
- **主动**：持续观察环境事件，主动推进值得处理的事，不需要时保持安静
- **持久**：始终在线，接入真实工作环境，异常时自动恢复

![四个基础特征](/images/2026-11-01-DeepSeek-Super-Agent/fig5-goals.webp "四个特征")

*连续、高效、主动、持久——单会话 Agent 一个都给不全。*

单个 ReAct Loop 撑不住这四点，底层需要一套架构。

### Router：常驻的全局调度器
Router 负责理解用户意图、管理全局上下文、分派任务、统一对外回复。它不直接执行任务，只负责协调，因此可以用轻量模型。

三个关键设计：

- **跨信道聚合**：IM、GitHub、邮件、CI 的消息先统一汇聚，Router 判断它属于哪个进行中的任务，还是需要触发新任务
- **订阅直通**：同一 Thread 的后续事件直接路由到已订阅的 Worker，不必经过 Router 重新判断，降低负载与延迟
- **事件合并**：短时间积压的多个事件合并进下一次调用一起处理，避免被高频消息淹没

### Worker：独立的执行单元
每个 Worker 拥有独立 Session，上下文彼此隔离，并行执行具体任务。Worker 用强模型，专心干一件事，结果交回 Router 统一回复。

- **隔离性**：窗口里只有当前任务相关信息，不会被其他任务的中间状态污染
- **生命周期管理**：任务完成后休眠或销毁；后续事件到达时从 Session 数据库恢复上下文、重新唤醒
- **Task 与 Session 解耦**：一个 Task 可能经历多个 Session（PR 被 review 后要改代码，Worker 休眠，评论到达再唤醒），但用户感知到的是同一个任务在推进

用户面对的始终是同一个 Agent，底层却能同时推进多个任务。Session 不再是用户需要管理的对象，而是系统内部的实现细节。

![Router + Worker 架构](/images/2026-11-01-DeepSeek-Super-Agent/fig2-router-worker.webp "双层架构")

*Router 只做语义路由与协调，Worker 各自带着独立上下文执行；状态统一落进 Session DB，随用随唤醒。*

这套结构与传统分布式系统的「调度器 + 工作节点」高度相似。DeepSeek 团队自己也承认：**相似的设计约束会导出相似的架构选择。**

## 事件驱动：从 Prompt 驱动走向环境监听
传统 Agent 等待用户输入；Super-agent 还能监听 IM 群聊、GitHub 评论、CI 结果、邮件，主动响应。做法是**两级过滤**：

- **Fast Path（规则过滤）**：用程序规则筛掉大部分噪音——无关关键词、非授权频道、已知无关的消息类型。这一步不调用模型，开销极低
- **Slow Path（模型判断）**：通过初筛的事件交给模型判断是否值得处理、属于哪个任务、要不要新建 Worker

核心思想是：**用廉价的规则处理大部分噪音，把昂贵的模型调用留给真正需要判断的情况。**

同一 Thread 的后续事件自动路由到已订阅的 Worker，意味着一个 PR 的评审评论可以直接唤醒对应 Worker 继续推进，不用从头再来。

文章里还有个细节：Super-agent 可以在用户还在讨论方案时，就提前尝试复现和修复问题；没有有效结果就保持安静。这体现了一个原则——**主动不等于打扰。**

![事件两级过滤](/images/2026-11-01-DeepSeek-Super-Agent/fig3-events.webp "事件过滤")

*规则层吸收噪音，模型层只处理需要判断的事件；同 Thread 的后续消息走订阅直通，绕开 Router。*

## 上下文管理：Inline & Spill 与 Chain of Recall
这是处理「多任务上下文爆炸」的核心机制。

### Inline & Spill：像 CPU 缓存一样管理上下文
- **Inline**：高频信息留在上下文内，相当于 L1/L2——当前任务的近期对话、关键决策点、待办事项
- **Spill**：低频信息溢出到外部存储（Session 数据库），相当于主存——历史任务摘要、已完成工作、跨任务关联信息
- **按需读取**：需要某段历史时从 Spill 加载到 Inline，而不是全量召回

这个设计同时解决了两个矛盾的目标：跨任务共享上下文（连续性），又避免不同任务的细节混在一起（隔离性）。

### Chain of Recall：沿着引用关系逐跳找回
需要历史上下文时，不是粗暴地「搜索所有历史」，而是沿着对象之间的引用关系逐跳找回。

比如一个 PR 评论提到「上次那个内存泄漏的问题」，Worker 可以通过引用关系找到之前处理该问题的 Task，再加载相关上下文。这比全文搜索更精准，也更省 token。

![上下文分层与召回](/images/2026-11-01-DeepSeek-Super-Agent/fig4-context.webp "上下文管理")

*Inline 放热数据，Spill 放冷数据；召回沿着引用链走，而不是全库扫描。*

## 异步交互、存算分离与 Watchdog
**异步全双工**：传统模式是「发指令 → 执行 → 返回」，用户在过程中被阻塞。Super-agent 把耗时任务放到后台，Agent 仍可响应新消息。Router 收到新消息后判断相关性：无关就新建 Worker 并行处理，相关则可以中断当前 Worker 或把消息排进队列。

**存算分离**：计算、持久化状态、执行环境三者解耦。推理在云端随时扩缩容；所有 Session 状态存在数据库里，不依赖执行环境；代码执行和工具调用可以在用户本地机器，通过轻量 daemon 与云端通信。这样即使本地关机，云端状态仍在，重新接入即可恢复——Agent 变成跨设备的连续实体，而不是绑定在某台机器上的进程。

![计算、状态、执行三层解耦](/images/2026-11-01-DeepSeek-Super-Agent/fig6-storage.webp "存算分离")

*云端负责推理，Session 数据库保存状态，本地 daemon 执行工具——关机不丢任务。*

**Watchdog**：独立的守护 Agent，不只监控进程存活，还检查 Worker 有没有陷入死循环、偏离目标，并在异常时尝试恢复。职责包括进度监控、死循环检测、偏离检测、异常恢复。这借鉴了操作系统里「看门狗定时器」的思想：执行单元本身可能出问题，需要一个外部观察者兜底。

![Watchdog 的四项职责](/images/2026-11-01-DeepSeek-Super-Agent/fig7-watchdog.webp "Watchdog")

*进度、死循环、偏离、恢复——监控的是健康度，不只是进程活着。*

## 效果：一组模拟数据
| 指标 | Super-agent | 单 Session 方案 |
|---|---|---|
| p50 完成时间 | 12.4 分钟 | 未给出 |
| p95 完成时间 | 15.7 分钟 | 29.5 分钟 |
| 缓存 token 开销 | 基准 | +27% |

三个观察：

- **p95 差距显著**：15.7 vs 29.5 分钟，长尾任务几乎快一倍，说明双层架构对长尾加速明显
- **p50 与 p95 差距小**：12.4 到 15.7 分钟，方差很小，完成时间更稳定
- **token 更省**：订阅直通让后续事件不必重新加载完整上下文，缓存 token 开销比单 Session 方案低约 27%

![延迟对比（模拟数据）](/images/2026-11-01-DeepSeek-Super-Agent/fig8-latency.webp "延迟")

*p95 从 29.5 分钟降到 15.7 分钟，且 p50 与 p95 差距很小。*

必须强调：这是**模拟实验数据**，不是生产环境指标。真实环境还有并发冲突、资源竞争、外部 API 限流等边界情况。

## 代价和局限
**架构复杂度显著增加。** 从单 Session 到 Router + Worker + Watchdog + 事件驱动 + 存算分离，复杂度翻了几倍：

- **调试**：一个任务涉及多个 Worker、多个 Session、多次唤醒休眠，复现问题比单 Session 难得多
- **运维**：要维护 Router 高可用、Worker 弹性扩缩容、Session 数据库持久化、Watchdog 独立部署
- **排查**：任务卡住时要依次检查 Router 分派、Worker 执行、Watchdog 监控、状态保存，路径长得多

**「主动」的边界很难定义。** 太安静，用户觉得它没干活；太主动，又变成骚扰。「没有有效结果则保持安静」这个原则本身就要模型判断什么是「有效结果」，而模型的判断可能和用户预期不一致。

**和 Redis 分片集群有什么区别？** 有评论质疑：这套架构本质就是「一个 key 对应一个 worker」。DeepSeek 也承认借鉴了经典系统设计思想。关键差异在于 Router 不是哈希路由，而要理解语义、判断任务归属、管理跨任务依赖——比算 hash 复杂得多。但另一个观点也值得记下：**成熟可靠的软件工程方式往往大同小异**，如果核心创新只是把分布式系统的经典思想搬进 Agent 领域，护城河可能没有想象中深。

**多轮演进未验证。** 文章验证的是单套设计的有效性，没有回答「这套结构能支撑多复杂的任务拓扑」，跨组织、跨权限、跨时区的场景仍是开放问题。

## 本质：从 Session 中心到 Goal 中心
这篇文章最大的价值不是五个模块的具体实现，而是一个方向性判断：**Agent 不应该围绕 Session 工作，而应该围绕 Goal 和环境事件持续运行。**

Session 是工具时代的产物——用户发一条指令，得到一条回复，对话结束。但真实工作从来不是这样：一个 PR 要改好几轮，一个 Bug 要在群里讨论半天，一个需求跨三个渠道推进。

Super-agent 的核心转变，是把 Agent 的组织单位从「对话」升级为「任务」，Session 降级为内部实现细节。

把它和之前的 ModularRSI 放在一起看很有意思：

- **ModularRSI** 解决「怎么让 Harness 自己变好」——向内优化
- **Super-agent** 解决「怎么让 Harness 同时扛住多个任务」——向外扩展

两者指向同一个结论：**模型决定能力上限，Harness 决定这些能力能否转化为持续生产力。**

一句话总结：**未来 Agent 的竞争焦点正在从「谁更聪明」转向「谁更可靠」——能持续、稳定、自主完成工作的系统，比只会聊天但需要人盯着的系统更有生产力价值。**

原文：<https://zhuanlan.zhihu.com/p/2099987320443237060>（DeepSeek《Super-agent 设计探索：迈向更自主的 Agent 系统》）

## Citation
Please cite this work as:

> Claire. "Agent Architecture: DeepSeek Super-agent and Multi-task Design". Claire-notes (Nov 2026). https://claire-notes.github.io/DeepSeek-Super-Agent

Or use the BibTeX citation:

```bibtex
@article{Claire-DeepSeekSuperAgent,
  title = {Agent Architecture: DeepSeek Super-agent and Multi-task Design},
  author = {Claire (qingYe)},
  journal = {claire-notes.github.io},
  year = {2026},
  month = {November},
  url = "https://claire-notes.github.io/DeepSeek-Super-Agent"
}
```
