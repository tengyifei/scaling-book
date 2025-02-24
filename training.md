---
layout: distill
title: "How to Parallelize a Transformer for Training"
# permalink: /main/
description: "Here we discuss four main parallelism schemes used during LLM training: data parallelism, fully-sharded data parallelism (FSDP), tensor parallelism, and pipeline parallelism. For each, we calculate at what point we become bottlenecked by communication."
date: 2025-02-04
future: true
htmlwidgets: true
hidden: false

section_number: 5

previous_section_url: "../transformers"
previous_section_name: "Part 4: Transformers"

next_section_url: ../applied-training
next_section_name: "Part 6: Training LLaMA"

bibliography: main.bib

giscus_comments: true

authors:
  - name: Jacob Austin
    url: "https://www.jacobaustin.org/"
    affiliations:
      name: Google DeepMind
  - name: Sholto Douglas
    url: "https://x.com/_sholtodouglas"
  - name: Roy Frostig
    url: "https://cs.stanford.edu/~rfrostig/"
  - name: Anselm Levskaya
    url: "https://anselmlevskaya.com/"
  - name: Charlie Chen
    url: "https://x.com/charliexychen"
  - name: Sharad Vikram
    url: "https://sharadvikram.com/"
  - name: Federico Lebron
    url: "https://fedelebron.com/"
  - name: Peter Choy
    url: "https://x.com/pchoy95"
  - name: Vinay Ramasesh
    url: "https://x.com/vinayramasesh"
  - name: Albert Webson
    url: "https://representation.ai/"
  - name: Reiner Pope<sup>*</sup>
    url: https://x.com/reinerpope

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - please use this format rather than manually creating a markdown table of contents.
toc:
  - name: "What Do We Mean By Scaling?"
  - subsections:
    - name: "Data Parallelism"
    - name: "Fully-Sharded Data Parallelism (FSDP)"
    - name: "Tensor Parallelism"
    - name: "Mixed FSDP and Tensor Parallelism"
    - name: "Pipelining"
    - name: "Scaling Between Pods"
  - name: "Takeaways from LLM Training on TPUs"
  - name: "Some Problems to Work"
  - name: "Appendix"

# Below is an example of injecting additional post-specific styles.
# This is used in the 'Layouts' section of this post.
# If you use this post as a template, delete this _styles block.
_styles: >
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
---

## What Do We Mean By Scaling?

The goal of “model scaling” is to be able to increase the number of chips used for training or inference while achieving a proportional, linear increase in throughput (we call this *strong scaling*). While performance on a single chip depends on the trade-off between memory bandwidth and FLOPs, performance at the cluster level depends on hiding inter-chip communication by overlapping it with useful FLOPS. This is non-trivial, because increasing the number of chips increases the communication load while reducing the amount of per-device computation we can use to hide it. As we saw in [Section 3](../sharding), sharded matrix multiplications often require expensive AllGathers or ReduceScatters that can block the TPUs from doing useful work. The goal of this section is to find out when these become *too expensive.*

In this section, we'll discuss four common parallelism schemes: (pure) **data parallelism, fully-sharded data parallelism** (FSDP / ZeRO sharding), **tensor parallelism** (also known as model parallelism), and (briefly) **pipeline parallelism**. For each, we'll show what communication cost we incur and at what point that cost starts to bottleneck our compute cost.<d-footnote>We'll focus on communication bounds — since while memory capacity constraints are important, they typically do not bound us when using rematerialization (activation checkpointing) and a very large number of chips during pre-training. We also do not discuss expert parallelism here for MoEs — which expands the design space substantially, only the base case of a dense Transformer.</d-footnote> For this section, you can focus solely on inter-chip communication costs, since as long as we have a large enough single-chip batch size, the transfer of data from HBM to MXU is already overlapped with computation.

We'll use the following notation to simplify calculations throughout this section.

| Notation | Meaning (model parameters)                                             |
| :------- | :--------------------------------------------------------------------- |
| D        | **d**<sub>model</sub> ( the hidden dimension/residual stream dim)      |
| F        | **d**<sub>ff</sub> (the feed-forward dimension)                        |
| B        | Batch dimension (number of tokens in the batch; total, not per-device) |
| T        | Sequence length                                                        |
| L        | Number of layers in the model                                          |

| Notation | Meaning (hardware characteristic)                                                                 |
| :------- | :------------------------------------------------------------------------------------------------ |
| C        | FLOPS/s per chip                                                                                  |
| W        | Network bandwidth (bidirectional, often subscripted as  e.g. $W_{\text{ici}}$ or $W_{\text{dcn}}$ |
| X        | Number of chips along mesh axis X                                                                 |
| Y        | Number of chips along an alternate mesh axis, labeled Y                                           |
| Z        | Number of chips along a third mesh axis, labeled Z                                                |

For simplicity's sake, **we'll approximate a Transformer as a stack of MLP blocks** — attention is a comparatively small fraction of the FLOPs for larger models as we saw in [Section 4](../transformers). We will also ignore the gating matmul, leaving us with the following simple structure for each layer:

{% include figure.liquid path="assets/img/simple-transformer.png" class="img-fluid" caption="<b>Figure:</b> a simplified Transformer layer. We treat each FFW block as a stack of two matrices <b>W<sub>in</sub></b>: <code>bf16[D, F]</code> (up-projection) and <b>W<sub>out</sub></b>: <code>bf16[F, D]</code> (down-projection) with an input <b>In</b>: <code>bf16[B, D]</code>." %}

Here are the 4 parallelism schemes we will discuss. Each scheme can be thought of as uniquely defined by a sharding for **In**, **W<sub>in</sub>, W<sub>out</sub>, and Out** in the above diagram.

**1. Data parallelism:** *activations sharded along batch, parameters and optimizer state are replicated on each device. Communication only occurs during the backwards pass.*

$$\text{In}[B_X, D] \cdot_D W_\text{in}[D, F] \cdot_F W_\text{out}[F, D] \rightarrow \text{Out}[B_X, D]$$

**2. Fully-sharded data parallelism (FSDP or ZeRO-3):** *activations sharded along batch (like pure data parallelism), parameters sharded along same mesh axis and AllGathered just-in-time before use in forward pass. Optimizer state also sharded along batch. Reduces duplicated memory.*

$$\text{In}[B_X, D] \cdot_D W_\text{in}[D_X, F] \cdot_F W_\text{out}[F, D_X] \rightarrow \text{Out}[B_X, D]$$

**3. Tensor parallelism (also called Megatron sharding or model parallelism):** *activations sharded along D ($d_\text{model}$), parameters sharded along F ($d_{ff}$). AllGather and ReduceScatter activations before and after each block. Compatible with FSDP.*

$$\text{In}[B, D_Y] \cdot_D W_\text{in}[D, F_Y] \cdot_F W_\text{out}[F_Y, D] \rightarrow \text{Out}[B, D_Y]$$

**4. Pipeline parallelism:** *weights sharded along the layer dimension, activations microbatched and rolled along the layer dimension. Communication between pipeline stages is minimal (just moving activations over a single hop). To abuse notation:*

$$\text{In}[L_Z, B, D][i] \cdot_D W_\text{in}[L_Z, D, F][i] \cdot_F W_\text{out}[L_Z, F, D][i] \rightarrow \text{Out}[L_Z, B, D_Y][i]$$

### Data Parallelism

**Syntax:** $$\text{In}[B_X, D] \cdot_D W_\text{in}[D, F] \cdot_F W_\text{out}[F, D] \rightarrow \text{Out}[B_X, D]$$

**When your model fits on a single chip with even a tiny batch size (>240 tokens, so as to be compute-bound), you should always use simple data parallelism.** Pure data parallelism splits our activations across any number of TPUs so long as the number of TPUs is smaller than our batch size. The forward pass involves no communication, but at the end of every step, each performs an **AllReduce on their gradients in order to synchronize them before updating the parameters.**

{% include figure.liquid path="assets/img/data-parallelism.png" class="img-fluid" caption="<b>Figure:</b> a diagram of pure data parallelism (forward pass). Our activations (left) are fully sharded along the batch dimension and our weights are fully replicated, so each TPU has an identical copy of the weights. This means the total memory of our weights is increased by a factor of N, but no communication is required on the forward-pass." %}

{% details Here's the full algorithm for the forward and backwards pass. We abuse notation to write dL/dOut as dOut, purely for compactness. %}

<div markdown=1 class="algorithm">

**Pure Data Parallelism Algorithm:**

**Forward pass:** need to compute Loss[B<sub>X</sub>]

1.  Tmp[B<sub>X</sub>, F] = In[B<sub>X</sub>, D] \*<sub>D</sub> W<sub>in</sub>[D, F]
2.  Out[B<sub>X</sub>, D] = Tmp[B<sub>X</sub>, F] \*<sub>F</sub> W<sub>out</sub>[F, D]
3.  Loss[B<sub>X</sub>] = ...

**Backward pass:** need to compute dW<sub>out</sub>[F, D], dW<sub>in</sub>[D, F]

1.  dOut[B<sub>X</sub>, D] = ...
2.  dW<sub>out</sub>[F, D] {U<sub>X</sub>} = Tmp[B<sub>X</sub>, F] \*<sub>B</sub> dOut[B<sub>X</sub>, D]
3.  dW<sub>out</sub>[F, D] = **AllReduce**(dW<sub>out</sub>[F, D] {U<sub>X</sub>}) (*not on critical path, can be done async*)
4.  dTmp[B<sub>X</sub>, F] = dOut[B<sub>X</sub>, D] \*<sub>D</sub> W<sub>out</sub>[F, D]
5.  dW<sub>in</sub>[D, F] {U<sub>X</sub>} = In[B<sub>X</sub>, D] \*<sub>B</sub> dTmp[B<sub>X</sub>, F]
6.  dW<sub>in</sub>[D, F] = **AllReduce**(dW<sub>in</sub>[D, F] {U<sub>X</sub>}) (*not on critical path, can be done async*)
7.  dIn[B<sub>X</sub>, D] = dTmp[B<sub>X</sub>, F] \*<sub>F</sub> W<sub>in</sub>[D, F] (*needed for previous layers*)

</div>

We ignore the details of the loss function and abbreviate $\text{Tmp} = W_\text{in} \cdot \text{In}$. Note that, although our final loss is the average **AllReduce**(Loss[B<sub>X</sub>]), we only need to compute the AllReduce on the backward pass when averaging weight gradients.

{% enddetails %}

Note that the forward pass has no communication — **it's all in the backward pass**! The backward pass also has the great property that the AllReduces aren't in the "critical path", meaning that each AllReduce can be performed whenever it's convenient and doesn't block you from performing subsequent operations. The overall communication cost _can still bottleneck us_ if it exceeds our total compute cost, but it is much more forgiving from an implementation standpoint. We'll see that model/tensor parallelism doesn't have this property.

**Why do this?** Pure data parallelism reduces activation memory pressure by splitting our activations over the batch dimension, allowing us to almost arbitrarily increase batch size as long as we have more chips to split the batch dimension over. Especially during training when our activations often dominate our memory usage, this is very helpful.

**Why not do this?** Pure data parallelism does nothing to reduce memory pressure from model parameters or optimizer states, which means pure data parallelism is rarely useful for interesting models at scale where our parameters + optimizer state don't fit in a single TPU. To give a sense of scale, if we train with parameters in bf16 and optimizer state in fp32 with Adam<d-footnote>Adam stores parameters, first order and second order accumulators. Since the params are in bfloat16 and optimizer state is in float32, this gives us `2 + 8 = 10` bytes per parameters.</d-footnote>, the largest model we can fit has $$\text{TPU memory} / 10$$ parameters, so e.g. on a TPUv5p pod with 96GB of HBM and pure data parallelism this is about 9B parameters.

<p markdown=1 class="takeaway">**Takeaway**: the largest model we can train with Adam and pure data parallelism has $$\text{num_params} = \text{HBM per device} / 10$$. For TPU v5p this is roughly 9B parameters.<d-footnote>Note that this doesn't include gradient checkpoints, so this wouldn't actually be useful. This is an absolute lower bound with a batch of 1 token.</d-footnote></p>

*To make this useful for real models during training, we'll need to at least partly shard the model parameters or optimizer.*

**When do we become bottlenecked by communication?** As we can see above, we have two AllReduces per layer, each of size $$2DF$$ (for bf16 weights). When does data parallelism make us communication bound? 

As in the table above, let $C$ = per-chip FLOPs, $W_{\text{ici}}$ = **bidirectional** network bandwidth, and $X$ = number of shards across which the batch is partitioned<d-footnote>We assume this partitioning is done over an ICI mesh, so the relevant network bandwidth is $W_\text{ici}$</d-footnote>.  Let's calculate the time required to perform the relevant matmuls, $$T_\text{math}$$, and the required communication time $$T_\text{comms}$$.  Since this parallelism scheme requires no communication in the forward pass, we only need to calculate these quantities for the backwards pass.

*Communication time:*  From a previous section we know that the time required to perform an AllReduce in a 1D mesh depends only on the total bytes of the array being AllReduced and the ICI bandwidth $W_\text{ici}$; specifically the AllReduce time is $2 \cdot \text{total bytes} / W_\text{ici}$. Since we need to AllReduce for both $W_\text{in}$ and $W_\text{out}$, we have 2 AllReduces per layer.  Each AllReduce is for a weight matrix, i.e. an array of $DF$ parameters, or $2DF$ bytes. Putting this all together, the total time for the AllReduce in a single layer is 

$$\begin{align}
T_\text{comms} &= \frac{2 \cdot 2 \cdot 2 \cdot D \cdot F}{W_\text{ici}}. \\
\end{align}$$

*Matmul time:* Each layer comprises two matmuls in the forward pass, or four matmuls in the backwards pass, each of which requires $2(B/X)DF$ FLOPs.  Thus

$$\begin{align}
T_\text{math} &= \frac{2 \cdot 2 \cdot 2 \cdot B \cdot L \cdot D \cdot F}{X \cdot C} \\
\end{align}$$

Since we overlap, the total time per layer is the max of these two quantities:

$$\begin{aligned}
T &\approx \max(\frac{8 \cdot B \cdot D \cdot F}{X \cdot C}, \frac{8 \cdot D \cdot F}{W_\text{ici}}) \\
T &\approx 8 \cdot D \cdot F \cdot \max(\frac{B}{X \cdot C}, \frac{1}{W_\text{ici}})
\end{aligned}$$

We become compute-bound when $$T_\text{math}/T_\text{comms} > 1$$, or when 

$$\begin{align}
\frac{B}{X} > \frac{C}{W_\text{ici}}.
\end{align}$$

The upshot is that, to remain compute-bound with data parallelism, we need the per-device batch size $$B / X$$ to exceed the ICI operational intensity, $C / W_\text{ici}$. This is ultimately a consequence of the fact that the computation time scales with the per-device batch size, while the communication time is independent of this quantity (since we are transferring model weights). Note the resemblance of the $B > C/W_\text{ici}$ condition to the single-device compute-bound rule $B > 240$; in that case as well, the rule came from the fact that computation time scaled with batch size while data-transfer size was (in the $B \ll F, D$ regime) independent of batch size.

Let's put in some real numbers to get a sense of scale. For TPUv5p, `C=4.6e14` and `W=2 * 9e10` for 1D data parallelism over ICI, so **our batch size per chip must be at least 2,550 to avoid being communication-bound**. Since we can do data parallelism over multiple axes, if we dedicate all three axes of a TPUv5p pod to pure data parallelism, we 3x our bandwidth $W_\text{ici}$ and can scale down to only BS=850 per TPU or 7.6M tokens per batch per pod (of 8960 chips)! **This tells us that it's fairly hard to become bottlenecked by pure data parallelism!**

<p markdown=1 class="takeaway">**Note on context parallelism:** throughout this section, we use $B$ to refer to the total batch size in tokens. Clearly, however, our batch is made up of $K$ sequences of $T$ tokens each, so how can we do this? As far as the MLP is concerned, *tokens are tokens*! It doesn't matter if they belong to the same batch or two different batches. So we are more or less free to do data parallelism over both the batch and sequence dimension: we call this context parallelism or sequence parallelism, but you can think of it as simply being another kind of data parallelism. Attention is trickier than the MLP since we do some cross-sequence computation, but this can be handled by gathering KVs or Qs during attention and carefully overlapping FLOPs and comms (typically using something called "ring attention"). Throughout this section, we will just ignore our sequence dimension entirely and assume some amount of batch or sequence parallelism.</p>


### Fully-Sharded Data Parallelism (FSDP)

**Syntax:** $$\text{In}[B_X, D] \cdot_D W_\text{in}[D_X, F] \cdot_F W_\text{out}[F, D_X] \rightarrow \text{Out}[B_X, D]$$

Fully-sharded data parallelism (often called FSDP or ZeRO-sharding<d-cite key="zero"></d-cite>) splits the model optimizer states and weights across the data parallel shards and efficiently gathers and scatters them as needed. **Compared to pure data parallelism, FSDP drastically reduces per-device memory usage and saves on backward pass FLOPs, with very minimal overhead.**

{% include figure.liquid path="assets/img/fsdp.png" class="img-fluid" caption="<b>Figure:</b> FSDP shards the contracting dimension of the MLP weights along the data dimension. This reduces memory but (from Section 3) requires us to gather the weights for W before we perform the matmul. Note that the activations (left) <it>are not sharded along the contracting dimension</it>, which is what forces us to gather. <b>Note that our weight optimizer state is likewise sharded along the contracting dimension.</b>" %}

You'll remember (from [Section 3](../sharding)) that an AllReduce can be decomposed into an AllGather and a ReduceScatter. This means that, instead of doing the full gradient AllReduce for standard data parallelism, we can shard the weights and optimizer states across chips, AllGather them at each layer during the forward pass and ReduceScatter across the weights during the backward pass at no extra cost. 

{% details Here's the full algorithm for FSDP. %}

<div markdown=1 class="algorithm">

**Fully-Sharded Data Parallelism (FSDP):**

**Forward pass:** need to compute Loss[B<sub>X</sub>]

1.  W<sub>in</sub>[D, F] = **AllGather**(W<sub>in</sub>[D<sub>X</sub>, F]) (*not on critical path, can do it during previous layer*)
2.  Tmp[B<sub>X</sub>, F] = In[B<sub>X</sub>, D] \*<sub>D</sub> W<sub>in</sub>[D, F] (*can throw away W<sub>in</sub>[D, F] now*)
3.  W<sub>out</sub>[F, D] = **AllGather**(W<sub>out</sub>[F, D<sub>X</sub>]) (*not on critical path, can do it during previous layer*)
4.  Out[B<sub>X</sub>, D] = Tmp[B<sub>X</sub>, F] \*<sub>F</sub> W<sub>out</sub>[F, D]
5.  Loss[B<sub>X</sub>] = ...

**Backward pass:** need to compute dW<sub>out</sub>[F, D<sub>X</sub>], dW<sub>in</sub>[D<sub>X</sub>, F]

1.  dOut[B<sub>X</sub>, D] = ...
2.  dW<sub>out</sub>[F, D] {U<sub>X</sub>} = Tmp[B<sub>X</sub>, F] \*<sub>B</sub> dOut[B<sub>X</sub>, D]
3.  dW<sub>out</sub>[F, D<sub>X</sub>] = **ReduceScatter**(dW<sub>out</sub>[F, D] {U<sub>X</sub>}) (*not on critical path, can be done async*)
4.  W<sub>out</sub>[F, D] = **AllGather**(W<sub>out</sub>[F, D<sub>X</sub>]) (*can be done ahead of time*)
5.  dTmp[B<sub>X</sub>, F] = dOut[B<sub>X</sub>, D] \*<sub>D</sub> W<sub>out</sub>[F, D] *(can throw away W<sub>out</sub>[F, D] here)*
6.  dW<sub>in</sub>[D,F] {U<sub>X</sub>} = dTmp[B<sub>X</sub>, F] \*<sub>B</sub> In[B<sub>X</sub>, D]
7.  dW<sub>in</sub>[D<sub>X</sub>, F] = **ReduceScatter**(dW<sub>in</sub>[D, F] {U<sub>X</sub>}) *(not on critical path, can be done async)*
8.  W<sub>in</sub>[D, F] = **AllGather**(W<sub>in</sub>[D<sub>X</sub>, F]) (*can be done ahead of time*)
9.  dIn[B<sub>X</sub>, D] = dTmp[B<sub>X</sub>, F] \*<sub>F</sub> W<sub>in</sub>[D, F] (*needed for previous layers) (can throw away W<sub>in</sub>[D, F] here*)

</div>

{% enddetails %}

This is also called "ZeRO Sharding", from "ZeRo Overhead sharding" since we don't perform any unnecessary compute or store any unnecessary state. ZeRO-{1,2,3} are used to refer to sharding the optimizer states, gradients, and weights in this way, respectively. Since all have the same communication cost, we can basically always do ZeRO-3 sharding, which shards the parameters, gradients, and optimizer states across a set of devices.

**Why would we do this?** Standard data parallelism involves a lot of duplicated work. Each TPU AllReduces the full gradient, then updates the full optimizer state (identical work on all TPUs), then updates the parameters (again, fully duplicated). For ZeRO sharding (sharding the gradients/optimizer state), instead of an AllReduce, you can ReduceScatter the gradients, update only your shard of the optimizer state, update a shard of the parameters, then AllGather the parameters as needed for your forward pass.

**When do we become bottlenecked by communication?** Our relative FLOPs and comms costs are exactly the same as pure data parallelism, since each AllReduce in the backward pass has become an AllGather + ReduceScatter. Recall that an AllReduce is implemented as an AllGather and a ReduceScatter, each with half the cost. Here we model the forward pass since it has the same FLOPs-to-comms ratio as the backward pass: 

$$\begin{aligned}
T_{math} &= \frac{2 \cdot 2 \cdot B \cdot D \cdot F}{X \cdot C} \\
T_{comm} &= \frac{2 \cdot 2 \cdot D \cdot F}{W_\text{ici}} \\
T &\approx \max\left(\frac{4 \cdot B \cdot D \cdot F}{X \cdot C}, \frac{4 \cdot D \cdot F}{W_\text{ici}}\right) \\
T &\approx 4 \cdot D \cdot F \cdot \max\left(\frac{B}{X \cdot C}, \frac{1}{W_\text{ici}}\right)
\end{aligned}$$

Therefore, as with pure data-parallelism, we are compute bound when $$B / X > C / W_\text{ici}$$, i.e. when the per-device batch size $B/X$ exceeds the "ICI operational intensity" $C/W_\text{ici}$ (`4.59e14 / 1.8e11 = 2550` for v5p). This is great for us, because it means if our per-device batch size is big enough to be compute-bound for pure data-parallelism, we can — without worrying about leaving the compute-bound regime — simply upgrade to FSDP, saving ourselves a massive amount of parameter and optimizer state memory!  Though we did have to add communication to the forward pass, this cost is immaterial since it just overlaps with forward-pass FLOPs. 

<p markdown=1 class="takeaway">**Takeaway:** both FSDP and pure data parallelism become bandwidth bound on TPUv5 when the batch size per device is less than $2550 / n_\text{axes}$.</p>

For example, DeepSeek-V2 (one of the only recent strong model to release information about its training batch size) used a batch size of ~40M tokens. **This would allow us to scale to roughly 47,000 chips, or around 5 TPUv5 pods, before we hit a bandwidth limit.**

For LLaMA-3 70B, which was trained for approximately `6.3e24 (15e12 * 70e9 * 6)` FLOPs, we could split a batch of 16M tokens over roughly `16e6 / (2550 / 3) = 18,823` chips (roughly 2 pods of 8960 chips), each with `4.59e14` FLOPs running at 50% peak FLOPs utilization (often called MFU), and **train it in approximately 17 days**. Not bad! But let's explore how we can do better.

<p markdown=1 class="takeaway">**Note on critical batch size**: somewhat unintuitively, we become more communication bottlenecked as our total batch size decreases (with fixed chip number). Data parallelism and FSDP let us scale to arbitrarily many chips so long as we can keep increasing our batch size! However, in practice, as our batch size increases, we tend to see diminishing returns in training since our gradients become almost noise-free. We also sometimes see training instability. Thus, the game of finding an optimal sharding scheme in the "unlimited compute regime" often starts from a fixed batch size, determined by scaling laws, and a known (large) number of chips, and then aims to find a partitioning that allows us to fit that small batch size on so many chips.</p>

### Tensor Parallelism

**Syntax:** $$\text{In}[B, D_Y] \cdot_D W_\text{in}[D, F_Y] \cdot_F W_\text{out}[F_Y, D] \rightarrow \text{Out}[B, D_Y]$$ (we use $$Y$$ to eventually combine with FSDP)

In a fully-sharded data-parallel AllReduce we move the weights across chips. We can also shard the feedforward dimension of the model and move the activations during the layer — this is called "1D model parallelism" or Megatron sharding<d-cite key="megatron"></d-cite>. This can unlock a smaller efficient batch size per pod. The figure below shows an example of a single matrix sharded in this way:

{% include figure.liquid path="assets/img/model-parallelism.png" class="img-fluid" caption="<b>Figure:</b> an example of basic tensor parallelism. Since we're only sharding our activations over Y (unlike in FSDP where we shard over X), we replicate our activations over X. Using our standard syntax, this is <b>A</b>[B, D<sub>Y</sub>] * <b>B</b>[D, F<sub>Y</sub>] -> <b>C</b>[B, F<sub>Y</sub>]. Because we're only sharding over one of the contracting dimensions, we typically AllGather the activations <b>A</b> before the matmul." %}

As noted, **In\[B, D<sub>Y</sub>\] \*<sub>D</sub> W<sub>in</sub>\[D, F<sub>Y</sub>\] \*<sub>F</sub> W<sub>out</sub>\[F<sub>Y</sub>, D\] \-\> Out\[B, D<sub>Y</sub>\] means we have to gather our activations before the first matmul. This is cheaper than ZeRO sharding when the activations are smaller than the weights.** This is typically true only with some amount of ZeRO sharding added (which reduces the size of the gather). This is one of the reasons we tend to mix ZeRO sharding and model parallelism.

{% details Here's the algorithm for tensor parallelism! %}

<div markdown=1 class="algorithm">

**Tensor Parallelism:**

**Forward pass:** need to compute Loss[B]

1.  In[B, D] = **AllGather**(In[B, D<sub>Y</sub>]) *(on critical path)*
2.  Tmp[B, F<sub>Y</sub>] = In[B, D] \*<sub>D</sub> W<sub>in</sub>[D, F<sub>Y</sub>] *(not sharded along contracting, so no comms)*
3.  Out[B, D] {U<sub>Y</sub>} = Tmp[B, F<sub>Y</sub>] \*<sub>F</sub> W<sub>out</sub>[F<sub>Y</sub>, D]
4.  Out[B, D<sub>Y</sub>] = **ReduceScatter**(Out[B, D] {U<sub>Y</sub>}) *(on critical path)*
5.  Loss[B] = ...

**Backward pass:** need to compute dW<sub>out</sub>[F<sub>Y</sub>, D], dW<sub>in</sub>[D, F<sub>Y</sub>]

1.  dOut[B, D<sub>Y</sub>] = ...
2.  dOut[B, D] = **AllGather**(dOut[B, D<sub>Y</sub>]) *(on critical path)*
3.  dW<sub>out</sub>[F<sub>Y</sub>, D] = Tmp[B, F<sub>Y</sub>] \*<sub>B</sub> dOut[B, D]
4.  dTmp[B, F<sub>Y</sub>] = dOut[B, D] \*<sub>D</sub> W<sub>out</sub>[F<sub>Y</sub>, D] *(can throw away dOut[B, D] here)*
5.  In[B, D] = **AllGather**(In[B, D<sub>Y</sub>]) *(this can be skipped by sharing with (1) from the forward pass)*
6.  dW<sub>in</sub>[D, F<sub>Y</sub>] = dTmp[B, F<sub>Y</sub>] \*<sub>B</sub> In[B, D]
7.  dIn[B, D] {U.Y} = dTmp[B, F<sub>Y</sub>] \*<sub>F</sub> W<sub>in</sub>[D, F<sub>Y</sub>] *(needed for previous layers)*
8.  dIn[B, D<sub>Y</sub>] = **ReduceScatter**(dIn[B, D] {U.Y}) *(on critical path)*

</div>

{% enddetails %}

One nice thing about tensor parallelism is that it interacts nicely with the two matrices in our Transformer forward pass. Naively, we would do an AllReduce after each of the two matrices. But here we first do **In[B, D<sub>Y</sub>] \* W<sub>in</sub>[D, F<sub>Y</sub>] -> Tmp[B, F<sub>Y</sub>]** and then **Tmp[B, F<sub>Y</sub>] \* W<sub>out</sub>[F<sub>Y</sub>, D] -> Out[B, D<sub>Y</sub>]**. This means we AllGather **In** at the beginning, and ReduceScatter **Out** at the end, rather than doing an AllReduce.

**How costly is this?** Let's only model the forward pass - the backwards pass is just the transpose of each operation here. In 1D model parallelism we AllGather the activations before the first matmul, and ReduceScatter them after the second, sending two bytes at a time (bf16). Let's figure out when we're bottlenecked by communication.

$$\begin{align}
T_{math} & = \frac{4 \cdot B \cdot D \cdot F}{Y \cdot C} \\
T_{comms} & =
\frac{2 \cdot 2 \cdot (B \cdot D)}{W_\text{ici}}\\
\textnormal{T} & \approx \max \left(\frac{4 \cdot B \cdot D \cdot F}{Y \cdot C}, \frac{2 \cdot 2 \cdot (B \cdot D)}{W_\text{ici}}\right)
\end{align}$$

Noting that we want compute cost to be greater than comms cost, we get:

$$\begin{align}
\frac{4 \cdot B \cdot D \cdot F}{Y \cdot C} > \frac{2 \cdot 2 \cdot (B \cdot D)}{W_\text{ici}}
\end{align}$$

$$\begin{align}
\frac{F}{Y \cdot C} > \frac{1}{W_\text{ici}}
\end{align}$$

$$\begin{align}
F > Y \cdot \frac{C}{W_\text{ici}}
\end{align}$$

Thus for instance, for TPUv5p, $$C / W_{ici} = 2550$$ in bf16, so we can only do tensor parallelism up to $$Y < F / 2550$$. When we have multiple ICI axes, our $$T_\text{comms}$$ is reduced by a factor of $n_\text{axes}$, so we get $$Y < n_\text{axes} * F / 2550$$.

<p markdown=1 class="takeaway">**Takeaway**: model parallelism becomes communication bound when $$Y > n_\text{axes} * F / 2550$$. For most models this is between 8 and 16-way model parallelism.</p>

**Note that this doesn't depend on the precision of the computation**, since e.g. for int8, on TPUv5p, $$C_\text{int8} / W_{ici}$$ is $$5100$$ instead of $$2550$$ but the comms volume is also halved, so the two factors of two cancel.

**Let's think about some examples:**

* On TPUv4p with LLaMA 3-70B with $$D = 8192,$$ $$F \approx 30,000$$, we can comfortably do 8-way model parallelism, but will be communication bound on 16 way model parallelism. The required F for model 8 way model sharding is 20k.

* For Gemma 7B, $$F \approx 50k$$, so we become communication bound with 19-way model parallelism. That means we could likely do 16-way and still see good performance.

### Mixed FSDP and Tensor Parallelism

**Syntax:** $$\text{In}[B_X, D_Y] \cdot_D W_\text{in}[D_X, F_Y] \cdot_F W_\text{out}[F_Y, D_X] \rightarrow \text{Out}[B_X, D_Y]$$

The nice thing about FSDP and tensor parallelism is that they can be combined. By sharding **W<sub>in</sub>** and **W<sub>out</sub>** along both axes we both save memory and compute. Because we shard B along X, we reduce the size of the model-parallel AllGathers, and because we shard F along Y, we reduce the communication overhead of FSDP. This means a combination of the two can get us to an even lower effective batch size than we saw above.

{% include figure.liquid path="assets/img/mixed-fsdp-model-parallelism.png" class="img-fluid" caption="<b>Figure:</b> a diagram combining FSDP and tensor parallelism. Unlike the other cases, there is no duplication of model parameters." %}

{% details Here's the full algorithm for mixed FSDP + tensor parallelism. While we have a lot of communication, all our AllGathers and ReduceScatters are smaller because we have batch-sharded our activations and tensor sharded our weights much more! %}

<div markdown=1 class="algorithm">

**Forward pass:** need to compute Loss[B]

1.  In[B<sub>X</sub>, D] = **AllGather**<sub>Y</sub>(In[B<sub>X</sub>, D<sub>Y</sub>]) *(on critical path)*
2.  W<sub>in</sub>[D, F<sub>Y</sub>] = **AllGather**<sub>X</sub>(W<sub>in</sub>[D<sub>X</sub>, F<sub>Y</sub>]) *(can be done ahead of time)*
3.  Tmp[B<sub>X</sub>, F<sub>Y</sub>] = In[B<sub>X</sub>, D] \*<sub>D</sub> W<sub>in</sub>[D, F<sub>Y</sub>]
4.  W<sub>out</sub>[F<sub>Y</sub>, D] = **AllGather**<sub>X</sub>(W<sub>out</sub>[F<sub>Y</sub>, D<sub>X</sub>]) *(can be done ahead of time)*
5.  Out[B<sub>X</sub>, D] {U.Y} = Tmp[B<sub>X</sub>, F<sub>Y</sub>] \*<sub>F</sub> W<sub>out</sub>[F<sub>Y</sub>, D]
6.  Out[B<sub>X</sub>, D<sub>Y</sub>] = **ReduceScatter**<sub>Y</sub>(Out[B<sub>X</sub>, D] {U.Y}) *(on critical path)*
7.  Loss[B<sub>X</sub>] = ...

**Backward pass:** need to compute dW<sub>out</sub>[F<sub>Y</sub>, D<sub>X</sub>], dW<sub>in</sub>[D<sub>X</sub>, F<sub>Y</sub>]

1.  dOut[B<sub>X</sub>, D<sub>Y</sub>] = ...
2.  dOut[B<sub>X</sub>, D] = **AllGather**<sub>Y</sub>(dOut[B<sub>X</sub>, D<sub>Y</sub>]) *(on critical path)*
3.  dW<sub>out</sub>[F<sub>Y</sub>, D] {U.X} = Tmp[B<sub>X</sub>, F<sub>Y</sub>] \*<sub>B</sub> dOut[B<sub>X</sub>, D]
4.  dW<sub>out</sub>[F<sub>Y</sub>, D<sub>X</sub>] = **ReduceScatter**<sub>X</sub>(dW<sub>out</sub>[F<sub>Y</sub>, D] {U.X})
5.  W<sub>out</sub>[F<sub>Y</sub>, D] = **AllGather**<sub>X</sub>(W<sub>out</sub>[F<sub>Y</sub>, D<sub>X</sub>]) *(can be done ahead of time)*
6.  dTmp[B<sub>X</sub>, F<sub>Y</sub>] = dOut[B<sub>X</sub>, D] \*<sub>D</sub> W<sub>out</sub>[F<sub>Y</sub>, D] *(can throw away dOut[B, D] here)*
7. In[B<sub>X</sub>, D] = **AllGather**<sub>Y</sub>(In[B<sub>X</sub>, D<sub>Y</sub>]) *(not on critical path + this can be shared with (2) from the previous layer)*
8.  dW<sub>in</sub>[D, F<sub>Y</sub>] {U.X} = dTmp[B<sub>X</sub>, F<sub>Y</sub>] \*<sub>B</sub> In[B<sub>X</sub>, D]
9.  dW<sub>in</sub>[D<sub>X</sub>, F<sub>Y</sub>] = **ReduceScatter**<sub>X</sub>(dW<sub>in</sub>[D, F<sub>Y</sub>] {U.X})
10. W<sub>in</sub>[D, F<sub>Y</sub>] = **AllGather**<sub>X</sub>(W<sub>in</sub>[D<sub>X</sub>, F<sub>Y</sub>]) *(can be done ahead of time)*
11. dIn[B<sub>X</sub>, D] {U.Y} = dTmp[B<sub>X</sub>, F<sub>Y</sub>] \*<sub>F</sub> W<sub>in</sub>[D, F<sub>Y</sub>] *(needed for previous layers)*
12. dIn[B<sub>X</sub>, D<sub>Y</sub>] = **ReduceScatter**<sub>Y</sub>(dIn[B<sub>X</sub>, D] {U.Y}) *(on critical path)*

</div>

{% enddetails %}

**What's the right combination of FSDP and MP?** A simple but key maxim is that FSDP moves weights and model parallelism moves activations. That means as our batch size shrinks (especially as we do more data parallelism), model parallelism becomes cheaper because our activations per-shard are smaller.

* Model parallelism performs $$\mathbf{AllGather}_Y([B_X, D_Y])$$ which shrinks as $$X$$ grows.  
* FSDP performs $$\mathbf{AllGather}_X([D_X, F_Y])$$ which shrinks as $$Y$$ grows.

Thus by combining both we can push our minimum batch size per replica down even more. We can calculate the optimal amount of FSDP and MP in the same way as above:

Let $$X$$ be the number of chips dedicated to FSDP and $$Y$$ be the number of chips dedicated to tensor parallelism. Let $$N$$ be the total number of chips in our slice with $$N=XY$$. Let $$M_X$$ and $$M_Y$$ be the number of mesh axes over which we do FSDP and MP respectively (these should roughly sum to 3). We'll purely model the forward pass since it has the most communication per FLOP. Then adding up the comms in the algorithm above, we have

$$T_\text{FSDP comms}(B, X, Y) = \frac{2\cdot 2\cdot D \cdot F}{Y \cdot W_\text{ici} \cdot M_X}$$

$$T_\text{MP comms}(B, X, Y) = \frac{2 \cdot 2 \cdot B \cdot D}{X \cdot W_\text{ici} \cdot M_Y}$$

And likewise our total FLOPs time is

$$T_\text{math} = \frac{2\cdot 2 \cdot B \cdot D \cdot F}{N \cdot C}.$$

To simplify the analysis, we make two simplifications: first, we allow $X$ and $Y$ to take on non-integer values (as long as they are positive and satisfy $XY=N$); second, we assume that we do not overlap comms on the $X$ and $Y$ axis. Under the second assumption, the total comms time is

$$T_\text{comms} = T_\text{FSDP comms} + T_\text{MP comms}.$$


Before we ask under what conditions we'll be compute-bound, let's find the optimal values for $X$ and $Y$ to minimize our total communication. Since our FLOPs is independent of $X$ and $Y$, the optimal settings are those that simply minimize comms. To do this, let's write $T_\text{comms}$ above in terms of $X$ and $N$ (which is held fixed, as it's the number of chips in our system) rather than $X$ and $Y$:

$$T_\text{comms} (X) = \frac{F \cdot X}{N \cdot M_X} + \frac{B}{X \cdot M_Y}$$

Differentiating this expression wrt $X$ and setting the derivative equal to zero gives the optimal value $X_{opt}$:

$$\begin{align*}
\frac{d}{dX} T_\text{comms} (X_{opt}) = \frac{F}{N \cdot M_X} - \frac{B}{X_{opt}^2 \cdot M_Y} \rightarrow \\
X_{opt} = \sqrt{\frac{B}{F} \frac{M_X}{M_Y} N}
\end{align*}$$

This is super useful! This tells us, for a given $B$, $F$, and $N$, what amount of FSDP is optimal. Let's get a sense of scale.  Plugging in realistic values, namely $N = 64$ (corresponding to a 4x4x4 array of chips), $B=48,000$, $F=32,768$, gives roughly $X\approx 13.9$. So we would choose $X$ to be 16 and $Y$ to be 4, close to our calculated optimum.

<p markdown=1 class="takeaway">**Takeaway:** in general, during training, the optimal amount of FSDP is $$X_{opt} = \sqrt{\frac{B}{F} \frac{M_X}{M_Y} N}$$. </p>

Now let's return to the question we've been asking of all our parallelism strategies: **under what conditions will we be compute-bound?** Since we can overlap FLOPs and comms, we are compute-bound when

$$T_\text{FSDP comms} + T_\text{MP comms} < T_\text{math}$$

which gives us

$$\frac{2\cdot 2\cdot D \cdot F}{Y \cdot W_\text{ici} \cdot M_X} + \frac{2 \cdot 2 \cdot B \cdot D}{X \cdot W_\text{ici} \cdot M_Y} < \frac{2\cdot 2 \cdot B \cdot D \cdot F}{N \cdot C}$$

Letting $\alpha \equiv C / W_\text{ici}$, the ICI arithmetic intensity, we can simplify:

$$\frac{F}{Y \cdot M_X} + \frac{B}{X \cdot M_Y} < \frac{B \cdot F}{N \cdot \alpha}$$

Plugging in our calculated $X_{opt}$ into the equation above (and noting that $Y_{opt} = N/X_{opt}$) results in the following condition on the batch size $B$:

$$ \sqrt{\frac{4 \cdot B\cdot F}{M_X \cdot M_Y \cdot N}} < \frac{B \cdot F}{N \cdot \alpha},$$

where the left-hand-side is proportional to the communication time and the right-hand-side is proportional to the computation time. Note that while the computation time scales linearly with the batch size (as it does regardless of parallelism), the communication time scales as the square root of the batch size. The ratio of the computation to communication time thus also scales as the square of the batch size:

$$ \frac{T_\text{math}}{T_\text{comms}} = \frac{\sqrt{BF}\sqrt{M_X M_Y}}{2\alpha \sqrt{N}}. $$

To ensure that this ratio is greater than one so we are compute bound, we require

$$ \frac{B}{N} > \frac{4\alpha^2}{M_X M_Y F}$$

See Appendix C for an alternate derivation of this relation. To get approximate numbers, again plug in $F=32,768$, $\alpha=2550$, and $M_X M_Y=2$ (as it must be for a 3D mesh). This gives roughly $B/N > 400$. This roughly wins us a factor of two compared to the purely data parallel (or FSDP) case, where assuming a 3D mesh we calculate that $B/N$ must exceed about $850$ to be compute bound.

<p markdown=1 class="takeaway">**Takeaway:** combining tensor parallelism with FSDP allows us to drop to a $B/N$ of $$2 \cdot 2550^2 / F$$. This lets us handle a batch of as little as 400 per chip, which is roughly a factor of two smaller than we could achieve with just FSDP.</p>

Below we plot the ratio of FLOPs to comms time for mixed FSDP + MP, comparing it both to only model parallelism and only data parallelism (FSDP), on a representative 4x4x4 chip array. While pure FSDP parallelism dominates for very large batch sizes, in the regime where batch size over number of chips is between roughly 400 and 850, a mixed FSDP + MP strategy is required in order to be compute-bound.

{% include figure.liquid path="assets/img/mixed-fsdp-comms-2.png" class="img-fluid" caption="<b>Figure:</b> ratio of FLOPs to comms time for optimal mixed FSDP/MP on a TPUv5p 4x4x4 slice with F=30k. As expected, model parallelism has a fixed ratio with batch size; ideal mixed FSDP + MP scales with $\sqrt{B}$, and FSDP scales with $B$. However, in intermediate batch size regimes, only FSDP + MP achieves a ratio greater than unity."%}

Here's another example of TPU v5p 16x16x16 showing the FLOPs and comms time as a function of batch size for different sharding schemes.

{% include figure.liquid path="assets/img/comms-flops-time.png" class="img-fluid" caption="<b>Figure:</b> time taken for communication with different parallelism schemes. The black dashed line is the time taken by the matrix multiplication FLOPs, so any curve above this line is comms-bound. We note that all strategies become comms-bound below batch size 1.5e6, which is in line with our expected 4096 * 2 * 2550^2 / (8192 * 4) = 1.6e10." %}

The black curve is the amount of time spent on model FLOPs, meaning any batch size where this is lower than all comms costs is strictly comms bound. You'll notice the black curve intersects the green curve at about `1.6e10`, as predicted. 

Zooming in, we can see that devoting two axes to FSDP, and using the optical switches to reconfigure the topology to have an 8-long axis for model sharding will give us the lowest communication volume between 1M and 6M batch size per slice, while pure FSDP combination is best between 6M and 100M. This agrees with our calculations above!

{% include figure.liquid path="assets/img/comms-flops-time-zoom.png" class="img-fluid" %}

Here's an interactive animation to play with this, showing the total compute time and communication time for different batch sizes:

<div class="l-page">
  <iframe src="{{ 'assets/plotly/training-roofline.html' | relative_url }}" frameborder='0' scrolling='no' height="400px" width="100%"></iframe>
</div>

You'll notice this generally agrees with the above (minimum around FSDP=256, MP=16), plus or minus some wiggle factor for some slight differences in the number of axes for each.

### Pipelining

You'll probably notice we've avoided talking about pipelining at all in the previous sections. Pipelining is a dominant strategy for GPU parallelism that is somewhat less essential on TPUs. Briefly, pipelined training involves splitting the layers of a model across multiple devices and passing the activations between pipeline stages during the forward and backward pass. The algorithm is something like:

1. Initialize your data on TPU 0 with your weights sharded across the layer dimension ($W_\text{in}[L_Z, D_X, F_Y]$ for pipelining with FSDP and tensor parallelism).
2. Perform the first layer on TPU 0, then copy the resulting activations to TPU 1, and repeat until you get to the last TPU.
3. Compute the loss function and its derivative $\partial L / \partial x_L$.
4. For the last pipeline stage, compute the derivatives $\partial L / \partial W_L$ and $\partial L / \partial x_{L-1}$, then copy $\partial L / \partial x_{L-1}$ to the previous pipeline stage and repeat until you reach TPU 0.

{% details Here is some (working) Python pseudo-code %}

This pseudocode should run on a Cloud TPU VM. While it's not very efficient or realistic, it gives you a sense how data is being propagated across devices.

```python
batch_size = 32
d_model = 128
d_ff = 4 * d_model

num_layers = len(jax.devices())

key = jax.random.PRNGKey(0)

# Pretend each layer is just a single matmul.
x = jax.random.normal(key, (batch_size, d_model))
weights = jax.random.normal(key, (num_layers, d_model, d_model)) 

def layer_fn(x, weight):
  return x @ weight

# Assume we have num_layers == num_pipeline_stages
intermediates = [x]
for i in range(num_layers):
  x = layer_fn(x, weights[i])
  intermediates.append(x)

  if i != num_layers - 1:
    x = jax.device_put(x, jax.devices()[i+1])

def loss_fn(batch):
  return jnp.mean(batch ** 2)  # make up some fake loss function

loss, dx = jax.value_and_grad(loss_fn)(x)

for i in range(0, num_layers, -1):
  _, f_vjp = jax.vjp(layer_fn, intermediates[i + 1], weights[i])
  dx, dw = f_vjp(dx)  # compute the jvp dx @ J(L)(x[i], W[i])
  weights[i] = weights[i] - 0.01 * dw  # update our weights

  if i != 0:
    dx = jax.device_put(dx, jax.devices()[i-1])
```

{% enddetails %}

**Why is this a good idea?** Pipelining is great for many reasons: it has a low communication cost between pipeline stages, meaning you can train very large models even with low bandwidth interconnects. This is often very useful on GPUs since they are not densely connected by ICI in the way TPUs are.

**Why is this difficult/annoying?** You might have noticed in the pseudocode above that TPU 0 is almost always idle! It's only doing work on the very first and last step of the pipeline. The period of idleness is called a pipeline bubble and is very annoying to deal with. Typically we try to mitigate this first with microbatching, which sends multiple small batches through the pipeline, keeping TPU 0 utilized for at least a larger fraction of the total step time.

A second approach is to carefully overlap the forward matmul $W_i @ x_i$, the backward $dx$ matmul $W_i @ \partial L / \partial x_{i+1}$, and the $dW$ matmul $\partial L / \partial x_{i+1} @ x_i$. Since each of these requires some FLOPs, we can overlap them to fully hide the bubble. Here's a plot from the recent DeepSeek v3 paper<d-cite key="DeepSeek3"></d-cite> showing their "bubble-free" pipeline schedule:

{% include figure.liquid path="assets/img/deepseek-pipeline.png" class="img-fluid" caption="<b>Figure:</b> the DeepSeek v3 pipeline schedule (from their <a href=\"https://github.com/deepseek-ai/DeepSeek-V3/blob/main/DeepSeek_V3.pdf\">recent paper</a>). Orange is the forward matmul, green is the dL/dx matmul, and blue is the dL/dW matmul. By prioritizing the backwards dL/dx multiplications, we can avoid \"stranding\" FLOPs." %}

Because it is less critical for TPUs (which have larger interconnected pods), we won't delve into this as deeply, but it's a good exercise to understand the key pipelining bottlenecks.

### Scaling Between Pods

Let's take a step back and look at a specific example, say training LLaMA-3 70B on TPU v5p. LLaMA-3 70B has $$F\approx 30,000$$. From the above sections, we know the following:

* We'll be ICI bound when we do model parallelism greater than $$Y > n_\text{axes} * F / 2550 \approxeq n_\text{axes} * 11$$. 
* Pure FSDP becomes ICI bound when we have a $$\text{batch size} < 2550 / n_\text{axes}$$. Here that means if we wanted to train with BS=2M, we'd at most be able to use $\approx 2400$ chips, which is roughly a quarter of a TPU v5p pod.
* Mixed FSDP + model parallelism becomes ICI bound when we have $$\text{batch size} < 2 \cdot 2550^2 / 30,000 = 432$$, so this lets us scale to roughly 9k chips! However, the maximum size of a TPU v5p pod is 8k chips, and beyond that we have to scale to lower-bandwidth data-center networking (DCN).

So this gives us a nice recipe to fit on a single pod with BS=3.5M. We'd use the equation above, which gives roughly X (FSDP) = 1024 and Y (MP) = 8. If the model was larger, there would be room to expand the model sharding to 16. We have a bit of room to drop the batch size as low as BS=1.5M on that pod and still be compute bound, but we're close to the lower bound there.

**To go larger than one pod, we need to scale over DCN.** Because DCN has lower bandwidth, it's typically too slow to do much useful FSDP. Instead, we do pure data parallelism over the DCN axis and FSDP within a pod. Lets calculate whether the Data Center Network (DCN) holds up.

With pure data parallelism over DCN, we need to sync the weights and optimizer states during each step (as the model completes its backward pass we need to complete the AllReduce). We can actually just borrow the math from the pure data parallelism section above which tells us that we become comms bound when the $\text{per pod batch size} < C_\text{pod} / W_\text{dcn}$ where the RHS here is the total compute and total bandwidth for the entire pod.

* Our total DCN ingress+egress bandwidth is 2.5e10 per host, with 4 chips per host. This gives us ~2000 hosts in the slice, and a total of `5e13` bytes of bandwidth.  
* $$C_\text{pod}$$ here is the pod size times the per-chip compute, which is `8k * 4.5e14 = 3.8e18` FLOPs.

As before, we become bottlenecked when $T_\text{math} < T_\text{comms}$ which happens when our $\text{per pod batch size} < C / W_\text{DCN} = 3.8e18 / 5e13 = 76,000$ (our pod level DCN operational intensity). For LLaMA-3, that's not going to be a problem since our per-pod batch size is much higher than that, but it could become an issue if we were to train on smaller slices (e.g. v5e). 

<p markdown=1 class="takeaway">**Takeaway:** This means we can scale fairly arbitrarily across pods, so e.g. with 10 pods of 8960 chips we could do a global batch size of about 40M tokens on 89,600 chips, training LLaMA-3 70B in about 2 days.</p>

## Takeaways from LLM Training on TPUs

* Increasing parallelism or reducing batch size both tend to make us more communication-bound because they reduce the amount of compute performed per chip.

* Up to a reasonable context length (~32k) we can get away with modeling a Transformer as a stack of MLP blocks and define each of several parallelism schemes by how they shard the two/three main matmuls per layer.

* During training there are 4 main parallelism schemes we consider, each of which has its own bandwidth and compute requirements (data parallelism, FSDP, model parallelism).

| **Strategy**                                 | **Description**                                                                                                                                                                            |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Data Parallelism**                         | Activations are batch sharded, everything else is fully-replicated, we all-reduce gradients during the backward pass.                                                                      |
| **FSDP**                                     | Activations, weights, and optimizer are batch sharded, weights are gathered just before use, gradients are reduce-scattered.                                                               |
| **Model Parallelism (aka Megatron, Tensor)** | Activations are sharded along $$d_\text{model}$$, weights are sharded along $$d_{ff}$$, activations are gathered before W<sub>in</sub>, the result reduce-scattered after W<sub>out</sub>. |
| **Mixed FSDP + Model Parallelism**           | Both of the above, where FSDP gathers the model sharded weights.                                                                                                                           |

And here are the "formulas" for each method:

$$\small
\begin{array}{cc}
\text{Strategy} & \text{Formula}\\
\hline
\text{DP} & \text{In}[B_X, D] \cdot_D W_\text{in}[D, F] \cdot_F W_\text{out}[F, D] \rightarrow \text{Out}[B_X, D] \\
\text{FSDP} & \text{In}[B_X, D] \cdot_D W_\text{in}[D_X, F] \cdot_F W_\text{out}[F, D_X] \rightarrow \text{Out}[B_X, D] \\
\text{MP} & \text{In}[B, D_Y] \cdot_D W_\text{in}[D, F_Y] \cdot_F W_\text{out}[F_Y, D] \rightarrow \text{Out}[B, D_Y] \\
\text{MP + FSDP}  & \text{In}[B_X, D_Y] \cdot_D W_\text{in}[D_X, F_Y] \cdot_F W_\text{out}[F_Y, D_X] \rightarrow \text{Out}[B_X, D_Y] \\
\hline
\end{array}$$

* Each of these strategies has a limit at which it becomes network/communication bound, based on their per-device compute and comms. Here's compute and comms per-layer, assuming $$X$$ is FSDP and $$Y$$ is model parallelism.

$$
\small
\begin{array}{ccc}
\text{Strategy} & \text{Compute per layer} & \text{Comms per layer} \\
& \text{(ignoring gating einsum)} & \text{(bytes, forward + backward pass)}\\
\hline
\text{DP} & 4BDF/X + 8BDF/X & 0 + 8DF \\
\text{FSDP} & 4BDF/X + 8BDF/X & 4DF + 8DF \\
\text{MP} & 4BDF/Y + 8BDF/Y & 4BD + 4BD \\
\text{FSDP + MP} & 4BDF/(XY) + 8BDF/(XY) & (4BD/X + 4DF/Y) + (8BD/X + 8DF/Y) \\
\hline
\end{array}$$

* Pure data parallelism is rarely useful because the model and its optimizer state use bytes = 10x parameter count. This means we can rarely fit more than a few billion parameters in memory.

* Data parallelism and FSDP become comms bound when the $$\text{batch size per shard} < C / W$$, the arithmetic intensity of the network. For ICI this is 2,550 and for DCN this is 75,000. This can be increased with more parallel axes.

* Model parallelism becomes comms bound when $$\lvert Y\rvert > F / 2550$$. **This is around 8-16 way for most models.** This is independent of the batch size.

* Mixed FSDP + model parallelism allows us to drop the batch size to as low as $$2 \cdot 2550^2 / F \approx 400$$. This is fairly close to the point (~200) where we become HBM bandwidth bound anyway.

* Data parallelism across pods requires a minimum batch size per pod of roughly 75,000 before becoming DCN-bound.

* Basically, if your batch sizes are big or your model is small, things are simple. You can either do data parallelism or FSDP \+ data parallelism across DCN. The middle section is where things get interesting.

## Some Problems to Work

Let's use LLaMA-2 13B as a basic model for this section. Here are some details:

| hyperparam              | value  |
| ----------------------- | ------ |
| n\_layers (L)           | 40     |
| d\_model (D)            | 5,120  |
| ffw\_multiplier (F / D) | 2.7    |
| n\_heads (N)            | 40     |
| n\_kv\_heads (K)        | 40     |
| d\_qkv (H)              | 128    |
| n\_embeddings (V)       | 32,000 |

**Question 1:** How many parameters does LLaMA-2 13B have (I know that's silly but do the math)? *Note that, as in [Transformer Math](../transformers), LLaMA-3 has 3 big FFW matrices, two up-projection and one down-projection. We ignored the two "gating" einsum matrices in this section, but they behave the same as W<sub>in</sub> in this section.*

{% details Click here for the answer. %}

* FFW parameters: $$3LDF$$ = `8.5e9`
* Attention parameters: $$4DNHL$$ = `4.2e9`
* Vocabulary parameters: $$2VD$$ = `0.3e9`
* Total: `8.5e9 + 4.2e9 + 0.39e9 = 13.1e9`, as expected!

{% enddetails %}

**Question 2:** Let's assume we're training with BS=16M tokens and using Adam. Ignoring parallelism for a moment, how much total memory is used by the model's parameters, optimizer state, and activations? *Assume we store the parameters in bf16 and the optimizer state in fp32 and checkpoint activations three times per layer (after the three big matmuls).*

{% details Click here for the answer. %}

The total memory used for the parameters (bf16) and the two optimizer states (fp32, the first and second moment accumulators) is `(2 + 4 + 4) * 13e9 ~ 130GB`. The activations after the first two matmuls are shaped $BF$ and after the last one $BD$ (per the Transformer diagram above), so the total memory for bf16 is $2 \cdot L \cdot (BD + 2 * BF) = 2LB \cdot (D + 2F)$ or `2 * 40 * 16e6 * 5,120 * (1 + 2 * 2.7) ~ 4.2e13 = 42TB`, since `B=16e16`. All other activations are more or less negligible.

{% enddetails %}

**Question 3:** Assume we want to train with 32k sequence length and a total batch size of 3M tokens on a TPUv5p 16x16x16 slice. Assume we want to use bfloat16 weights and a float32 optimizer, as above.

1. Can we use pure data parallelism? Why or why not? 
2. Can we use pure FSDP? Why or why not? With pure FSDP, how much memory will be used per device (assume we do gradient checkpointing only after the 3 big FFW matrices). 
3. Can we use mixed FSDP + model parallelism? Why or why not? If so, what should $X$ and $Y$ be? How much memory will be stored per device? Using only roofline FLOPs estimates and ignoring attention, how long will each training step take?

{% details Click here for the answer. %}

First, let's write down some numbers. With 32k sequence length and a 3M batch size, we have a sequence batch size of 96. On a TPU v5p 16x16x16 slice, we have `393TB` of HBM.

1. We can't use pure data parallelism, because it replicates the parameters and optimizer states on each chip, which are already around 130GB (from Q2) which is more HBM than we have per-chip (96GB).

2. Let's start by looking purely at memory. Replacing BS=16M with 3M in Q2, we get `~7.86e12` total checkpoint activations, and with the 1.3e11 optimizer state this brings us to almost exactly 8e12 = 8TB. The TPUv5p slice has `393TB` of HBM in total, so we are safely under the HBM limit. Next let's look at whether we'll be comms or compute-bound. With 4096 chips and 3 axes of parallelism, we can do a minimum batch size of `850 * 4096 = 3.48M` tokens. That's slightly above our 3M batch size. So we're actually comms-bound, which is sad. So the general answer is **no, we cannot do FSDP alone**.

3. Now we know our primary concern is being comms-bound, so let's plug in some numbers. First of all, from the discriminant above, we know our per-chip batch size with mixed FSDP + model parallelism needs to be above $2 \cdot 2550^2 / F = 940$ here, which is actually slightly worse than pure FSDP. Obviously that's sort of an artifact of some of the approximations we made, but this suggests mixed FSDP + model parallelism isn't actually much better. Partly this is because $F$ is so small we can't do a full axis worth of model parallelism. One way around this is to do small subrings of 4 chips of tensor parallelism and dedicate the remaining bandwidth of the first axis to FSDP. We won't do the math out but it's good to check that we probably can do this without being comms-bound.
 
{% enddetails %}

**Question 4:** What if we wanted to drop to batch size 1M? How does this affect the answers to question 3? What about batch size 10M?

<h3 markdown=1 class="next-section">That's it for Part 5! For Part 6, which applies this content to real LLaMA models, [click here](../applied-training)!</h3>

## Appendix

### Appendix A - More stuff about FSDP

Here's a nice extra figure showing how FSDP shards parameters/gradients. The rows are, in order, pure data parallelism, ZeRO-1/2/3. There's not much reason not to do ZeRO-3 since it has effectively the same communication load.

{% include figure.liquid path="assets/img/fsdp-figure.png" class="img-fluid" %}

**Figure:** diagram showing parameter, gradient, and optimizer state memory for pure data parallelism, ZeRO-1/2/3 respectively. [Source](https://arxiv.org/abs/1910.02054).

### Appendix B - Deriving the comms necessary for the backward passes

Above, we simplified the Transformer layer forward pass as Out\[B, D\] \= In\[B, D\] \*D W<sub>in</sub>\[D, F\] \*<sub>F</sub> W<sub>out</sub>\[F, D\] . How do we derive the comms necessary for the backwards pass?

This follows fairly naturally from the rule in the previous section for a single matmul **Y = X \* A**:

$$\frac{dL}{dA} = \frac{dL}{dY}\frac{dY}{dA} = X^T \left(\frac{dL}{dY}\right)$$

$$\frac{dL}{dX} = \frac{dL}{dY}\frac{dY}{dX} = \left(\frac{dL}{dY}\right) A^T$$

Using this, we get the following formulas (letting Tmp\[B, F\] stand for In\[B, D\] \* W<sub>in</sub>\[D, F\]):

<div markdown=1 class="algorithm">

1. dW<sub>out</sub>[F, D] = Tmp[B, F] \*<sub>B</sub> dOut[B, D] 
2. dTmp[B, F] = dOut[B, D] \*<sub>D</sub> W<sub>out</sub>[F, D] 
3. dW<sub>in</sub> = dTmp[B, F] \*<sub>B</sub> Tmp[B, F] 
4. dIn[B, D] = dTmp[B, F] \*<sub>F</sub> W<sub>in</sub>[D, F]

</div>

Note that these formulas are mathematical statements, with no mention of sharding.  The job of the backwards pass is to compute these four quantities.  So to figure out the comms necessary, we just take the shardings of all the quantities which are to be matmulled in the four equations above (Tmp, dOut, W<sub>out</sub>, W<sub>in</sub>), which are specified by our parallelization scheme, and use the rules of sharded matmuls to figure out what comms we have to do.  Note that dOut is sharded in the same way as Out.

### Appendix C - Alternate derivation of the batch size constraint for mixed FSDP + model parallelism

Above we derived that when using a combination of FSDP + model parallelism, we can be compute-bound when 

$$ \frac{B}{N} > \frac{4\alpha^2}{M_X M_Y F} $$

Here we present an alternate derivation of this fact.  We start by setting the communication time equal to the computation time, and look for a condition which makes this equality impossible.

$$\frac{F}{Y \cdot M_X} + \frac{B}{X \cdot M_Y} = \frac{B \cdot F}{N \cdot \alpha}$$

Since $XY=N$, we can rewrite in terms of $X$:

$$\frac{FX}{N \cdot M_X} + \frac{B}{X \cdot M_Y} = \frac{B \cdot F}{N \cdot \alpha}$$, or

$$X^2 \frac{F}{N \cdot M_X} + \frac{B}{M_Y} - X \frac{B \cdot F}{N \cdot \alpha} = 0.$$

As this is a quadratic in $X$, the point at which we'll have no solutions is the point at which the discriminant becomes zero.  This occurs when

$$B^2\cdot F^2 \cdot M_X^2 \cdot M_Y^2 - 4\cdot \alpha^2 \cdot F \cdot B \cdot N \cdot M_Y \cdot M_X = 0$$

or by simplifying

$$B\cdot F \cdot M_X \cdot M_Y - 4\cdot \alpha^2 \cdot N = 0$$

which gives us

$$B = \frac{4 \cdot \alpha^2 \cdot N}{F \cdot M_X \cdot M_Y}$$

so our total batch size divided by the total number of chips cannot drop below

$$\frac{4 \alpha^2}{F \cdot M_X \cdot M_Y},$$

as we had derived above.
