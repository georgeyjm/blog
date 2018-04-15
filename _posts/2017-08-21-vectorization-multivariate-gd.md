---
layout:      post
title:       'A Note on the Vectorization of Multivariate Gradient Descent'
subtitle:    ''
date:        2017-08-21 22:36:47
author:      'George Yu'
header-img:  'img/blog-default.jpg'
tags:
    - Machine Learning
---

The algorithm for a step of multivariate gradient descent is commonly defined as to update
\\[ \theta_j := \theta_j - \alpha \frac{\partial}{\partial\theta_j}J(\theta) \\]
for all \\(j = (0\dots n)\\) simultaneously, where \\(n\\) is the number of features, \\(\alpha\\) is the learning rate,
and \\(\frac{\partial}{\partial\theta_j}J(\theta)\\) is the partial derivative of the cost function \\(J\\) with respect to \\(\theta_j\\).

Calculating the partial derivative, we get
\\[ \frac{\partial}{\partial\theta_j}J(\theta) = \frac{1}{m} \sum_{i=1}^m(h_\theta(X^{(i)}) - y^{(i)})X_j^{(i)} \\]

where \\(m\\) is the number of training examples, \\(h_\theta\\) is the hypothesis function with the specific \\(\theta\\), \\(x^{(i)}\\) and \\(y^{(i)}\\) denotes the input (features) and output (target value) of the \\(i\\)<sup>th</sup> training example repectively.

In this case, the training input matrix \\(X \in \mathbb{R}^{m\times n+1}\\) has a column for each parameter, and a row for each input values,
or that
\\[
X =
\begin{bmatrix}
  X_0^{(1)} & X_1^{(1)} & \dots & X_n^{(1)} \\\\\\
  X_0^{(2)} & X_1^{(2)} & \dots & X_n^{(2)} \\\\\\
  \vdots & \vdots & \ddots & \vdots \\\\\\
  X_0^{(m)} & X_1^{(m)} & \dots & X_n^{(m)} \\\\\\
\end{bmatrix}
\\]
and the training output vector \\(y \in \mathbb{R}^m\\) contains target outputs for all \\(m\\) training examples.

Apparently, using a `for` loop to compute and update the \\(\theta_j\\) values is not efficient,
so the vectorization of this multvariate gradient descent becomes very important.

First, instead of letting \\(\theta\\) be multiple parameters \\(\theta_0,\ \theta_1, \dots, \theta_n\\), let \\(\theta\\) be a \\((n + 1)\\)-th dimension vector,
such that
\\[
\theta =
\begin{bmatrix}
  \theta_0 \\\\\\
  \theta_1 \\\\\\
  \vdots \\\\\\
  \theta_n
\end{bmatrix}
\\]
this way, while \\(\theta_i\\) still refers to the \\(i\\)<sup>th</sup> parameter, the vectorization process becomes much easier.

Then, define one step of gradient descent as
\\[ \theta := \theta - \alpha\delta \\]
where \\(\delta \in \mathbb{R}^{n+1}\\) denotes the vector of all \\(\theta_j\\) changes,
such that \\(\delta_j = \frac{\partial}{\partial\theta_j}J(\theta)\\) for all \\(j = (0\dots n)\\), upon simplification, we get
\\[
\begin{align}
\delta_j &= \frac{\partial}{\partial\theta_j}J(\theta) \\\\\\
         &= \frac{1}{m} \sum_{i=1}^m(h_\theta(X^{(i)}) - y^{(i)})X_j^{(i)} \\\\\\
         &= \frac{1}{m} \sum_{i=1}^m(X^{(i)}\theta - y^{(i)})X_j^{(i)} \\\\\\
\end{align}
\\]
Notice the second part is the sum of \\(m\\) products, so we should be able to write it as the product of two matrices,
\\(a \in \mathbb{R}^{1\times m}\\) and \\(b \in \mathbb{R}^m\\), each defined as,
\\[ a_{1,i} = X^{(i)}\theta - y^{(i)} \\]
\\[ b_i = X_j^{(i)} \\]
thus, the equation for \\(\delta_j\\) can be further simplified to
\\[ \delta_j = \frac{1}{m} (X\theta - y)^\top X_j \\]
Since \\(\frac{1}{m} (X\theta - y)^\top\\) is constant with respect to the variable \\(j\\), we can extract it out and the define the vector \\(\delta\\) as
\\[
\begin{align}
  \delta &= \frac{1}{m} (X\theta - y)^\top
  \begin{bmatrix}
    X_0 & X_1 & \dots & X_n
  \end{bmatrix}
\end{align}
\\]

Therefore, we can finally vectorize the equation for \\(\delta\\) as
\\[ \delta = \frac{1}{m} (X\theta - y)^\top X \\]
or, applying the rules of matrix transpose operation,
\\[ \delta = \frac{1}{m}\ X^\top (X\theta - y) \\]
