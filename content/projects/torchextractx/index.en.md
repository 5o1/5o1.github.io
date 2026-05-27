---
title: "TorchExtraContext"
date: 2026-05-19
draft: false
description: "A utility for passing loss, metric, and log information out of deep networks."
summary: "A utility for passing loss, metric, and log information out of deep networks."
tags: ["PyTorch", "MoE", "Deep Learning", "Tooling"]
categories: ["Projects"]
ShowToc: true
TocOpen: false
hidemeta: false
comments: false
searchHidden: false
weight: 10
---

Last year, I worked on a Mixture of Experts (MoE) project. MoE models often need an additional load-balancing loss on top of the task loss. From a quick look at common PyTorch patterns, when people need to return multiple losses, they usually put them into a dict in `forward` and pass that dict outward layer by layer. That gets annoying very quickly.

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

If I kept using this dict-passing approach, I would have to modify every Module along the bottom-up path.

So I built `TorchExtraContext` for my own needs. The pipeline is roughly:

1. Create a Context object.
2. Choose a root Module and register the Context reference on each named Module. I designed it this way so the context can be reached from nested modules.
3. Inside each Module's `forward`, call the API to register loss, metric, or log entries on the Context.
4. In the root Module or training loop, fetch all loss, metric, and log values from the root Module's Context.

The project is available on [GitHub](https://github.com/5o1/TorchExtraContext).

I do not currently need TorchScript support, and although I have not tested it, this tool probably does not support TorchScript.
