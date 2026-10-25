---
layout: post
title: Harnessed Agentic RL (01)
---

### Harnessed Agentic RL训练的几个困难
1.Agent = Harness + Model，问题在于：每换一个Harness框架，传统Agent RL训练的框架就要改一遍。改 harness 代码、适配 API 格式、处理 KV cache、对齐 token 序列……工作量巨大，而且 harness 一更新就容易断。
