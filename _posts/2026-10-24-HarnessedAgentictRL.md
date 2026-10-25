---
layout: post
title: Harnessed Agentic RL (01)
---

#### Harnessed Agentic RL训练的几个困难
> Agent = Harness + Model, Agent RL训练难点主要有
>> 1. Harness框架越来越庞大复杂，已经无法写入GPU Clauster，训练与推理环境偏差
>> 2. 每更换一个Harness框架，Agent RL训练框架就要随之改一遍（harness 代码、适配 API 格式、处理 KV cache、对齐 token 序列…）
>> 3. 不同Harness的训练数据如何复用，任意Harness无损接入RL训练
***
