---
title: "TorchExtraContext"
date: 2026-05-19
draft: false
description: "用来从深层网络传递loss/metric/log的工具。"
summary: "用来从深层网络传递loss/metric/log的工具。"
tags: ["PyTorch", "MoE", "Deep Learning", "Tooling"]
categories: ["Projects"]
ShowToc: true
TocOpen: false
hidemeta: false
comments: false
searchHidden: false
weight: 10
---

去年我做了一个 MoE (Mixture of Experts) 相关的工作。MoE 需要在任务损失之外，再引入负载均衡损失。我简单查了下，面对“返回多个损失”需求，PyTorch 里常见的做法是把损失放在 forward 的 dict 返回值中，再一层层向外传递。这非常麻烦！

```python
def block(x, losses):
losses["aux_loss"] = x.abs().mean()
return x, losses

def model(x):
losses = {}
x, losses = block1(x, losses)
x, losses = block2(x, losses)
return x, losses

x, losses = model(x)
total_loss = losses["task_loss"] + 0.1 * losses["aux_loss"]
```

如果继续沿用通过 dict 逐层传递方案，那么我需要沿 Bottom-up 路径修改所有的Modules！

所以我根据我的需求编写了`TorchExtraContext`。它的管线大致是这样的：

1. 创建一个Context对象。
2. 选择一个根Module，向每个named Module注册Context的引用。我设计了
3. 在Module的forward里，调用API向Context注册loss/metric/log。
4. 在根Module（training loop）里，从根Module的Context里获取所有的loss/metric/log。

该项目可以通过[GitHub Repo](https://github.com/5o1/TorchExtraContext)获取。

另外由于目前我还没有Torch Script的使用需求，虽然我还没测试过，不过这个工具大概率不支持Torch Script。