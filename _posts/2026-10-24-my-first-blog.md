---
layout: post
title: Harnessed Agentic RL (01)
---

#### Harnessed Agentic RL训练的几个困难
众所周知 Agent = Harness + Model
现阶段的Agentic RL训练问题在于：
1.Harness框架越来越庞大复杂，已经无法写入GPU Clauster，直接带来训练与推理环境偏差
2.每测试或更换一个Harness框架，Agent RL训练框架就要随之改一遍。改 harness 代码、适配 API 格式、处理 KV cache、对齐 token 序列……工作量巨大
3.不同Harness的训练数据无法复用，训练代价大
