---
title: 使用數學符號
date: 2022-08-12 09:00:00
tags:
  - KaTeX
  - Math
categories:
  - notes
keywords:
  - KaTeX
math: KaTeX
---

## 使用KaTeX

### 行內、區塊

When \\(a \ne 0\\), there are two solutions to \\(ax^2 + bx + c = 0\\) and they are

<!--more-->

$$
x = {-b \pm \sqrt{b^2-4ac} \over 2a}
$$

### 羅倫茲方程

$$
\begin{aligned}
\dot{x} & = \sigma(y-x) \\\
\dot{y} & = \rho x - y - xz \\\
\dot{z} & = -\beta z + xy
\end{aligned}
$$

### 柯西－史瓦茲不等式

$$
\left( \sum_{k=1}^n a_k b_k \right)^2 \leq \left( \sum_{k=1}^n a_k^2 \right) \left( \sum_{k=1}^n b_k^2 \right)
$$

### 向量外積

$$
\mathbf{V}_1 \times \mathbf{V}_2 =  \begin{vmatrix}
\mathbf{i} & \mathbf{j} & \mathbf{k} \\\
\frac{\partial X}{\partial u} &  \frac{\partial Y}{\partial u} & 0 \\\
\frac{\partial X}{\partial v} &  \frac{\partial Y}{\partial v} & 0
\end{vmatrix}
$$

### 投擲 *n* 個硬幣得到 *k* 個人頭的機率

$$
P(E) = {n \choose k} p^k (1-p)^{ n-k}
$$

### An Identity of Ramanujan

$$
\frac{1}{\Bigl(\sqrt{\phi \sqrt{5}}-\phi\Bigr) e^{\frac25 \pi}} =
1+\frac{e^{-2\pi}} {1+\frac{e^{-4\pi}} {1+\frac{e^{-6\pi}}
{1+\frac{e^{-8\pi}} {1+\ldots} } } }
$$

### A Rogers-Ramanujan Identity

$$
1 +  \frac{q^2}{(1-q)}+\frac{q^6}{(1-q)(1-q^2)}+\cdots =
\prod_{j=0}^{\infty}\frac{1}{(1-q^{5j+2})(1-q^{5j+3})},
\quad\quad \text{for $|q|<1$}.
$$

### 馬克士威方程

$$
\begin{aligned}
\nabla \times \vec{\mathbf{B}} -\ \frac1c\ \frac{\partial\vec{\mathbf{E}}}{\partial t} & = \frac{4\pi}{c}\vec{\mathbf{j}} \\\\[1.0em]
\nabla \cdot \vec{\mathbf{E}} & = 4 \pi \rho \\\\[0.5em]
\nabla \times \vec{\mathbf{E}}\ +\ \frac1c\ \frac{\partial\vec{\mathbf{B}}}{\partial t} & = \vec{\mathbf{0}} \\\\[1.0em]
\nabla \cdot \vec{\mathbf{B}} & = 0 \end{aligned}
$$

Inline math: \\(\varphi = \dfrac{1+\sqrt5}{2}= 1.6180339887…\\)

Block math:
$$
 \varphi = 1+\frac{1} {1+\frac{1} {1+\frac{1} {1+\cdots} } }
$$
