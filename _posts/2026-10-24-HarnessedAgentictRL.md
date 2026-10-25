---
layout: post
title: Harnessed Agentic RL (01)
---

###### 今天来聊下Harnessed Agentic RL的Agent LightningV1.0  
众所周知：Agent = LLM + Harness.  
Modern agents do not operate as standalone LLMs. They run inside agent harnesses that manage tools, context, and control flow, which makes the harness a critical component. How to directly involve the deploy-time harness into model RL-training, thereby narrowing the gap between training and actual use is the term harnessed agentic RL.

>> 1. Harness框架越来越庞大复杂，已经无法写入GPU Clauster，训练与推理环境偏差
>> 2. 更换Harness框架，Agent RL训练框架就要随之改一遍（harness 代码、适配 API 格式、处理 KV cache、对齐 token 序列…）
>> 3. 不同Harness框架的训练数据如何复用，任意Harness无损接入RL训练

###### 传统Agentic-RL框架 VS Harnessed-Agentic-RL框架

###### 一些挑战 

###### 一些实验结果比对


