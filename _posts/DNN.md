---
layout: post
title:  "Literature Review: A Doubly Decoupled Network for Edge Detection (DNN)"
typora-root-url: ../
category: Literature Review
tag: [computer vision, deeplearning]
toc: true
math: true
#author_profile: false
---

# **A Doubly Decoupled Network for Edge Detection (DNN)**

## 1. Introduction – What’s This Paper About?

Edge detection might sound fancy, but it’s basically about finding the outlines or boundaries in an image—like the edges of a chair, the shape of a face, or where one object ends and another begins. It’s super important for a bunch of computer vision tasks like object detection, segmentation, and image editing.

### The Problem with Existing Edge Detectors

Most modern edge detection models work by asking a simple question for each pixel: “Is this part of an edge or not?” That sounds reasonable, but there are a couple of big problems:

1. **Too few edge pixels:**  
   Most of an image is *not* edge—maybe only 5% of pixels actually belong to edges. So the model sees a whole lot of “no edge” and just a tiny bit of “yes edge.” That imbalance makes it really easy for the model to overfit or memorize noise instead of learning useful patterns.

2. **One model trying to do too much:**  
   A single model usually tries to capture both **fine details** (like sharp edges) *and* **big-picture understanding** (like what object this edge belongs to). That’s like trying to zoom in and out at the same time—not easy!

### What This Paper Proposes

This paper introduces a new model called **DDN (Doubly Decoupled Network)** that takes a smarter approach:

1. **Two brains are better than one:**  
   Instead of one big encoder doing everything, they use two:
   - A *spatial encoder* that focuses on small details (where the edge is),
   - A *semantic encoder* that looks at the bigger picture (what the edge means).

2. **Don’t just say yes or no—give a probability:**  
   Rather than saying “this pixel *is* or *isn’t* an edge,” DDN lets the model say something like:  
   *“I’m 80% sure this is an edge, but here’s how confident I am.”*  
   It does this by modeling edges using a **Gaussian distribution**, which is just a fancy way of saying it predicts a range of possible values instead of just one.

### Why This Is Cool

- It helps the model *not* overfit to tiny edge labels.
- It creates smoother, more natural-looking outlines—especially in tricky areas like shadows or textured backgrounds.
- It works better even with smaller datasets.

---

> 🎯  So in short: DDN makes edge detection smarter by splitting up the tasks and being more honest about uncertainty. It’s a more human way to think about edges—sometimes they’re obvious, sometimes they’re fuzzy. And this model gets that.

---

## 2. Related Work – How This Paper Builds on the Past

To really get why DDN is cool, we need to understand what other researchers have done before—and where things still felt a bit off. Let’s go section by section.

---
### 2.1. Edge Detection – From Classic to Deep Learning

Edge detection is all about figuring out where the meaningful boundaries are in an image—like where your cat ends and your blanket begins.

**Old-school methods** like Canny or Sobel used handcrafted filters. They were fast but easily confused by lighting, texture, or shadows.

Then deep learning arrived, and the game changed.

Models like:
- **HED** brought in deep supervision (teaching all layers of the network),
- **RCF** and **BDCN** stacked up features from different levels,
- And more recent methods like **DiffusionEdge** and **EDTER** used attention and transformer-like modules.

These models improved accuracy and edge sharpness, but they all shared a common limitation: they still treated edge detection as a **hard binary task**—predicting 0 or 1 per pixel.

---

### 2.2. The Overfitting Problem in Edge Detection

Here’s the catch: edge labels are **super sparse**.

> On average, only about 5% of an image’s pixels are actual edges.

So what happens? Models tend to:
- Overfit (memorize) those few edge pixels,
- Ignore or mislabel uncertain or fuzzy regions,
- Get really good at the training set but stumble on new data.

Even fancy methods like deep supervision or class balancing can’t fully solve the problem—they’re working *around* the issue, not *through* it.

---

### 2.3. Learnable Gaussian Distribution – A Smarter Way to Think About Edges

Instead of saying “yes or no” for every pixel, wouldn’t it make more sense to say:

> “I’m 85% sure this is an edge... and I feel kinda confident about that.”

That’s where the **learnable Gaussian distribution** comes in.
- DDN doesn’t predict a fixed value.
- It predicts a **mean (μ)** and a **standard deviation (σ)** for each pixel.
- Then it samples edge maps from that distribution.

This adds a little bit of **randomness**, which helps:
- Make the model less likely to memorize noise,
- Capture uncertainty (some edges *are* fuzzy),
- Improve generalization to new images.

It’s like giving the model some room to say “I think it’s an edge, but I’m not totally sure”—which is way more realistic.

---

### 2.4. Bilateral Network – Splitting the Work Makes It Smarter

Another big idea from past research is the **Bilateral Network**, which splits the feature extractor into two branches:

- One for **spatial details** (like sharp edges and fine lines),
- One for **semantic meaning** (like “this is part of a face”).

Why does this help?
- Because trying to do both with one encoder is like using a microscope and a telescope at the same time. It’s just too much.
- By decoupling the features, each branch can focus on what it’s good at.

DDN builds on this idea and pushes it further:
- It uses **two encoders**, and keeps them separate.
- Then it **fuses** the results to get the best of both worlds: sharp edges that make sense.

---

> 🎯 In short: DDN takes the best lessons from the past—deep learning, uncertainty modeling, and feature splitting—and combines them into something more flexible, more reliable, and more human-like in how it detects edges.

---

## 3. Proposed Methods

### 3.1. Formulation – How Does DDN Actually Work?

So far, we've talked about how DDN is different because it doesn’t just say "yes" or "no" for each edge—it says **how likely** a pixel is to be part of an edge, and **how confident** it is about that guess.

Here’s how that works under the hood (don’t worry, we’ll keep it simple).

---

#### The Usual Way: Binary Labels

Most edge detectors use **binary labels** for supervision:
- If a pixel is part of an edge → label = 1  
- If not → label = 0

Then they use a **binary cross-entropy loss** to train the model.

But remember the problems:
- Edge pixels are rare.
- Binary labels are too harsh for fuzzy or uncertain edges.

---

#### The DDN Way: Predicting a Distribution

Instead of predicting just a number like 0.9 (edge) or 0.1 (non-edge), DDN predicts a **distribution** over possible values.

Specifically, it predicts:
- A **mean** ($\mu$) — "How likely is this pixel to be an edge?"
- A **standard deviation** ($\sigma$) — "How confident am I about that prediction?"

These two values define a **Gaussian (bell-shaped) curve**.  
That’s fancy math speak for: “Let’s assume the edge-ness of this pixel is not a fixed value, but a range with a center and a spread.”

From that curve, we sample values to generate **probabilistic edge maps**.

---

#### The Key formula

The key idea:  
> The predicted edge value for each pixel is sampled like this:

$$
\hat{y} \sim \mathcal{N}(\mu, \sigma^2)
$$

Where:
- $\hat{y}$: predicted edge score for a pixel  
- $\mu$: predicted mean (how likely it’s an edge)  
- $\sigma$: predicted standard deviation (uncertainty)

Then the final edge map is made by averaging multiple samples.

---

#### Why This Is Smart

- **Reduces overfitting**: Instead of memorizing hard labels, the model explores different possibilities.
- **Handles fuzzy edges**: Some edges are soft or partially visible. This method captures that uncertainty.
- **Feels more human**: We don’t always make hard decisions—we estimate and adapt. This model does the same.

---

> So instead of forcing the model to say “this *is* an edge,” DDN lets it say “this *might* be an edge... and here’s how sure I am.” That’s a big mindset shift—and a very effective one.

---

### 3.2. Network Structure – Two Brains Are Better Than One

Now let’s look under the hood of DDN and see how it’s built.

Instead of cramming everything into one big encoder like traditional models, DDN uses **two separate branches**:
- One that focuses on **fine spatial details** like tiny contours and sharp lines.
- Another that captures **semantic meaning** like object boundaries and high-level structure.

Then, it blends the results using a smart probabilistic approach.

---

#### Breaking Down Figure 1 – What’s What in the Architecture

*Figure 1: DDN's architecture includes two encoders, dual decoders, and a Gaussian sampler that produces flexible edge maps.*

Here’s your visual decoder for the figure:

| 🧩 Diagram Part | 📌 Component Name | 💡 What It Does |
|----------------|------------------|-----------------|
| 🟦 Top-left blue blocks (`Conv`, `2×down`) | **Spatial Encoder** | Extracts fine-grained details and outlines while keeping resolution high. |
| 🟧 Bottom-left orange blocks (`SeparConvs`, `SAs`) | **Semantic Encoder** | Learns high-level, object-aware context using separable convolutions and self-attention. |
| 🟩 Green & Yellow `IRB` blocks | **Decoder Blocks (IRBs)** | Gradually upsamples and merges features from the encoders. Used in both decoders. |
| 🟪 Output to $\hat{\mu}$ | **Mean Decoder** | Predicts edge intensity—how likely each pixel is part of an edge. |
| 🟦 Output to $\hat{\sigma}^2$ | **Variance Decoder** | Predicts uncertainty—how confident the model is about each pixel being an edge. |
| 🎲 $\mathcal{N}(\hat{\mu}, \hat{\sigma}^2)$ block | **Gaussian Sampler** | Samples multiple edge maps from the predicted distribution. |
| 🖤 Final output image | **Edge Prediction** | Probabilistic edge map after sampling and sigmoid activation. |
| 🔄 Right-side stacked block | **Meta Block** | Core building block used throughout the network. Combines normalization, operators (Conv or Self-Attention), and MLP layers. |

---

#### Step-by-Step: What’s Going On?

##### 1️⃣ Spatial Encoder (🟦 Blue path)
This branch focuses on **precise visual details**—edges, lines, and textures.

- It’s lightweight and uses a couple of convolution layers.
- It doesn't go too deep because its job is to preserve spatial precision.
- The output feeds into the **mean decoder**, which estimates edge strength.

##### 2️⃣ Semantic Encoder (🟧 Orange path)
This branch captures **semantic understanding**—what kind of object we're looking at and where edges *should* be.

- Uses **Separable Convolutions** to save compute and **Self-Attention (SAs)** at the end for global context.
- The deeper layers understand what the scene contains.
- Its output feeds into the **variance decoder**, which estimates uncertainty.

> 💡 Why split them up? Because one path is focused on *where* things are sharp, while the other is focused on *why* those things matter. It's a divide-and-conquer strategy.

##### 3️⃣ Dual Decoders with IRB Blocks (🟩)
After encoding, the network uses two decoders—one for $\mu$ (mean), and one for $\sigma^2$ (variance).

- These decoders are built with stacks of **IRBs (Inverted Residual Blocks)**.
- They gradually upsample the encoded features and refine the output edge maps.

Each decoder learns something different:
- The **mean decoder** estimates how "edgy" each pixel is.
- The **variance decoder** estimates how "uncertain" that edge is.

##### 4️⃣ Gaussian Sampler 🎲

This is where the magic happens.

Instead of outputting a hard decision, DDN says:

> “Here’s how sure I am about this edge, and here’s how confident I feel about that prediction.”

It treats each pixel’s edge score as a **Gaussian distribution**:

$$
\hat{y} \sim \mathcal{N}(\mu, \sigma^2)
$$

- Then it **samples multiple edge maps** from this distribution.
- Finally, it averages them to create a **smooth and flexible final edge map**.

---

#### TL;DR – Why This Design Works

- **Two separate encoders** = each part of the network focuses on what it does best.
- **Gaussian modeling** = better at dealing with fuzziness and edge ambiguity.
- **Decoders + sampling** = less overfitting, more human-like decisions.

---

> 🔍 Final thoughts: DDN is designed to work like our own visual system—paying attention to both details and big-picture structure, while also accepting uncertainty instead of forcing binary answers. And that’s what makes it special.

---

### 3.3. Training & Inference – How DDN Learns (and Predicts)

We’ve seen how DDN is built—but how does it actually **learn** to detect edges?  
And once it’s trained, how does it **make predictions**?

This section breaks it down.

---

#### During Training: Teaching the Model

Just like any other deep learning model, DDN needs a loss function to learn from its mistakes. But instead of just one loss, DDN uses two:

| 💥 Loss Type                                          | 🔍 What It Measures                                           |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| **Edge Loss** ($\mathcal{L}_{\text{edge}}$)          | How well the predicted edge map matches the ground truth.    |
| **Normalization Loss** ($\mathcal{L}_{\text{norm}}$) | How close the predicted distribution is to a "nice" standard shape (a normal distribution). |

The final loss is:

$$
\mathcal{L}_{\text{total}} = \mathcal{L}_{\text{edge}} + \phi \cdot \mathcal{L}_{\text{norm}}
$$

Where:
- $\phi$ is a hyperparameter that controls how much weight we give to regularization.

---

##### 1. Edge Loss: Handling Imbalanced Data

Remember: **most pixels are NOT edges**—maybe 95% of them!

So if we use a standard loss, the model might get lazy and just say "no edge" everywhere.  
To avoid this, DDN **balances the edge vs. non-edge pixels** using a *weighted cross-entropy loss*.

That helps the model focus more on the rare but important edge pixels.

---

##### 2. Normalization Loss: Stay in Shape!

DDN doesn’t just predict edges—it predicts a **distribution** (mean $\mu$ and variance $\sigma^2$).  
We want this distribution to stay well-behaved (not too wild), so we **regularize** it using **KL divergence**.

This keeps the learned distribution close to a standard bell curve $\mathcal{N}(0, 1)$, which improves generalization and reduces overfitting.

---

#### During Inference: Making Predictions

Here’s the cool part.

Instead of just doing one forward pass and getting a final result, DDN:

1. **Predicts $\mu$ and $\sigma$** for each pixel.
2. **Samples multiple edge maps** from the distribution:
   $$
   \hat{y}_i \sim \mathcal{N}(\mu, \sigma^2)
   $$
3. **Passes each sample through a sigmoid** to get values between 0 and 1.
4. **Averages all the samples** to produce the final edge map.

This is kind of like asking the model:

> "Can you guess what the edge might look like—multiple times—and then average your answers?"

---

#### How Many Samples?

The paper tried different numbers (1, 5, 10, 100, 500, even 1000).  
Turns out, **500 samples** hit the sweet spot:
- Enough to remove randomness,
- Not too slow for practical use.

That’s why DDN is a bit heavier at inference time—but the gain in quality is worth it, especially for tricky, noisy, or low-contrast edges.

---

#### TL;DR – How DDN Learns and Predicts

| Stage | What Happens |
|-------|---------------|
| **Training** | Learns from both pixel-wise edge accuracy and distribution regularity. |
| **Inference** | Samples many edge maps from predicted distributions and averages them. |
| **Result** | Clean, flexible, and less overfitted edges that work even in complex scenes. |

---

> So instead of trying to be perfect in one shot, DDN learns to be *probabilistically honest*—saying “here’s what I think, here’s how sure I am,” and then taking the average. That’s a lot like how humans process uncertain visual info.

---

### 4. Experiments – Does DDN Actually Work?

Now that we know how DDN is designed and trained, let’s talk about how well it performs.

This section covers:
- Which datasets were used
- How the model was trained
- And most importantly: how it compares to other state-of-the-art models

---

#### Datasets Used

DDN was tested on **three standard edge detection benchmarks**:

| Dataset | Description |
|---------|-------------|
| **BSDS500** | Natural scenes with hand-drawn edge labels (classic benchmark) |
| **NYUDv2** | Indoor scenes, includes RGB + depth information |
| **BIPED** | High-quality outdoor images with very clean, single-pixel edge labels |

---

#### Training Details (Quick Facts)

- Optimizer: Adam  
- Batch size: 4  
- Learning rate: 1e-4 (or 1e-5 if pretrained)  
- Number of samples during inference: 500  
- Loss: Cross-entropy + KL divergence (as explained in Section 3.3)

---

#### Results: How Did It Perform?

| Model | ODS (BSDS500) | Params | GFLOPs |
|-------|----------------|--------|--------|
| HED | 0.788 | 14.6M | 57.5 |
| RCF | 0.798 | 14.8M | 75.3 |
| BDCN | 0.806 | 16.3M | 103.4 |
| **DDN (ours)** | **0.842** | 41.2M | 71.2 |

- 📌 **ODS (Optimal Dataset Scale)** is a common edge detection accuracy metric.
- DDN outperforms all previous models on BSDS500, even those with much higher complexity.

Also, on **NYUDv2** and **BIPED**, DDN achieves:
- **Smoother edges**
- **Better generalization**
- **More robust performance in complex scenes**

---

#### Visual Results

In challenging examples (like blurry objects or textured backgrounds), DDN produces cleaner and more accurate outlines compared to others.

---

> In short: DDN isn’t just a cool idea on paper—it actually works better than previous methods across multiple datasets, while staying relatively efficient in terms of computation.

---

### 5. Ablation Study – What Makes DDN Actually Good?

The authors ran a series of tests to answer a simple question:

> “Which parts of DDN are really making a difference?”

They changed or removed parts of the model to see what happens to performance.

---

#### Key Findings

| ✅ Component | 🔄 What If We Remove or Change It? | 📉 Result |
|-------------|-----------------------------------|-----------|
| **Dual Encoder** | Use just one encoder (no decoupling) | Accuracy drops → each branch adds value |
| **Gaussian Sampling** | Remove the distribution and just predict a single edge map | Less flexible, more overfitting |
| **Variance Decoder** | Only use mean, ignore uncertainty | Performance drops, especially in complex regions |
| **Transformer in Semantic Encoder** | Replace with only CNNs | Slight performance drop (less context awareness) |

---

#### Summary

- Both **spatial and semantic encoders** are necessary—they specialize in different things.
- Predicting a **distribution** instead of a fixed edge map helps avoid overfitting.
- The **uncertainty prediction (variance)** is useful, especially for fuzzy or low-contrast edges.
- Adding **self-attention** (in the semantic path) improves results, especially on complex scenes.

---

> TL;DR: DDN’s design choices aren’t just fancy—they each play a clear role. When you take them out, the performance drops. That’s solid evidence that the "double decoupling" idea works.

---

### 🧾 6. Conclusion – Why DDN Matters

So, what have we learned from DDN?

At its core, DDN is built on a simple but powerful idea:

> 👉 Don’t treat edge detection as a fixed yes/no problem—treat it as a **distribution** problem.  
> 👉 Don’t force one encoder to do everything—**decouple** the tasks and specialize.

This "double decoupling" approach—one encoder for detail, one for meaning, and one decoder each for predicting mean and variance—helps DDN:
- Reduce overfitting
- Handle low-contrast or noisy edges
- Generalize better across datasets

The results speak for themselves:
- SOTA performance on BSDS500 and other benchmarks ✅
- More robust edge maps, even in tough conditions ✅
- A fresh take on what it means to “detect” an edge ✅

---

> 🎯 In short: DDN isn’t just a better model—it’s a smarter way to think about edges.

Whether you’re working on art generation, medical imaging, or AR filters, DDN's probabilistic approach offers a flexible and powerful edge detection solution.

---





