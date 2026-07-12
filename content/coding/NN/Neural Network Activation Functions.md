---
title: Neural Network Activation Functions
draft: false
tags:
  - nn
  - ai
created: 2026-03-11 19:56
modified: 2026-07-12 00:00
---

## 1. Neuron Structure

A standard neuron in modern neural networks can be written as:
$$
y = \sigma(Wx + b)
$$
where

- $W$ : weight matrix
- $x$ : input vector
- $b$ : bias
- $\sigma$ : activation function

The neuron consists of two parts:

1. **Linear transformation**

$$Wx + b$$

2. **Activation function**

$$\sigma(\cdot)$$

---

## 2. Why Activation Functions Are Necessary

If we remove activation functions:

$$y = W_2(W_1x + b_1) + b_2$$

This can be rewritten as:

$$y = W'x + b'$$

Therefore:

**Stacking multiple linear layers is still just a linear transformation.**

So neural networks need activation functions to introduce **non-linearity**.

---

## 3. Core Roles of Activation Functions

Activation functions affect what a network can represent, how it trains, and how an output is interpreted.

### 3.1 Introduce Non-Linearity

With activation functions:

$$f(x) = \sigma(W_3 \, \sigma(W_2 \,  \sigma(W_1x)))$$

The network can approximate complex nonlinear functions.

This is related to the **Universal Approximation Theorem**, which states that a neural network with nonlinear activation can approximate any continuous function.

---

### 3.2 Enable Gradient-Based Optimization

Neural networks are trained using **backpropagation**.

We need gradients such as:

$$\partial L / \partial W$$

Therefore activation functions must be:

- differentiable
- numerically stable

Examples:

| Function | Issue |
|--------|------|
|sigmoid | vanishing gradient |
|tanh | still small gradient |
|ReLU | stable gradient |

This is why modern networks often use:

- ReLU
- GELU
- Swish

---

### 3.3 Sparsity / Gating Effect

Some activation functions create **sparse activations**.

Example:

ReLU
$$
\text{ReLU}(x) = \max(0, x)
$$
If input < 0 → output = 0.

This effectively **turns off neurons**.

Benefits:

- sparse representation
- implicit feature selection
- improved efficiency

---

### 3.4 Control Output Distribution

Activation functions are also used in the **output layer** to match the task.

Examples:

#### Binary classification

sigmoid

output range:

\[0, 1\]

interpreted as probability.

---

#### Multi-class classification

softmax

$$p_i = \exp(z_i) / \sum \exp(z_i)$$

Properties:

- probability distribution
- sum = 1

---

#### Regression

Different tasks use different outputs:

| Task                 | Activation      |
| -------------------- | --------------- |
| unbounded regression | identity        |
| range \[-1,1\]       | tanh            |
| positive values      | ReLU / softplus |

---

## 4. Modern Perspective

A neural network can be viewed as:

$$f(x) = \sum w_i \phi_i(x)$$

where $\phi_i$ are **basis functions**.

Activation functions determine the **shape of these basis functions**.

Example:

ReLU networks approximate functions using **piecewise linear functions**.

This explains why deep networks with simple activations can still approximate complex functions.

---

## 5. Why Transformers / LLMs Use Simple Activations

Modern large models usually use:

- ReLU
- GELU

Reasons:

### 5.1 Deep Networks Create Complexity

Even simple activation functions can produce complex functions when stacked across many layers.

Depth dramatically increases expressive power.

---

### 5.2 Piecewise Linear Geometry

ReLU networks approximate functions using many **linear regions**.

Deep networks increase the number of these regions exponentially.

---

### 5.3 Optimization Stability

ReLU/GELU provide better training dynamics:

- stable gradients
- efficient computation
- good empirical performance

---

## 6. Practical Defaults

For hidden layers, ReLU is the classic default and GELU is common in Transformers / LLMs. For the output layer, choose an activation from the task: sigmoid for binary classification, softmax for multi-class classification, and identity for regression.
