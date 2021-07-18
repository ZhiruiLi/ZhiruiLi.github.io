---
layout: post
title: 图形学入门：坐标变换
excerpt:
categories: articles
author: zrl
date: 2021-07-17
modified: 2021-07-17
tags:
  - Computer Graphics
comments: true
share: true
---

# 概述

将一个物体显示到屏幕上，这个事情似乎非常简单，以至于我们基本上认为它已经天经地义到直接告诉计算机我们要显示什么物体它就会自动显示出来，毕竟我们拍照的时候就是举起相机按下快门就会出现一张图片了。但事实上，相机是基于物理感光元件实现了从三维世界到二维图片的投影，在计算机的程序世界中一切都需要被计算出来，也就是说，我们只有一堆图形的描述信息，我们需要自己将这些图形在二维的平面上绘制的方式告诉操作系统，操作系统才能最终在屏幕上绘制出我们想要的图形。

那么，我们究竟要进行怎样的一些计算呢？我们可以将这个过程和拍照进行类比，物体的位置、角度，相机的位置、角度以及相机本身设置的一些参数都会对拍照的结果产生影响，相机离物体近，物体就显得大一些，相机往左偏，物体在最终相片上的位置就会往右。显然，光有场景中物体本身的模型信息还不足以让我们知道最终呈现在屏幕上的图像的样子，我们还需要考虑上述的种种信息才能最终得出在二维的平面上这个场景最终的形态，这些计算主要分为三部分：

- 模型空间到世界空间的变换

  这个过程将物体的每个顶点坐标从自己模型空间移动到世界空间，也就是将物体移动到世界的对应位置摆放好。

- 世界空间到观察空间的变换

  这个过程将物体的每个顶点坐标从世界空间移动到相机的观察空间，由于位置的移动是相对的，这也就相当于把相机移动到对应位置摆放好。只不过为了计算方便，我们一般假设相机的位置就在原点的位置，看向 z 轴负方向。

- 观察空间到裁剪空间的变换

  这个过程就是将物体的每个顶点坐标从三维空间投影到相机的二维成像平面上，这也就相当于相机拍照时在胶片上记录下当时的画面。

# 数学基础

为了说明这三种变换在计算机中是如何进行的，这里需要先补充一点相关的基础知识。在计算机中，为了进行快速的计算，采用了矩阵（Matrix）这一数学工具。下面是一个 $$3 \times 2$$ 的矩阵（即 $$3$$ 行 $$2$$ 列的矩阵）：

$$
A =
\begin{bmatrix}
  1 & 2 \\
  3 & 4 \\
  5 & 6
\end{bmatrix}
$$

矩阵有一个操作叫转置（Transpose），矩阵 $$A$$ 的转置写作 $$A^\mathrm{T}$$，这个过程其实就是将矩阵沿着左上到右下的对角线翻转，即把 $$A$$ 的每一行写 $$A^\mathrm{T}$$ 的列，把 $$A$$ 的每一列写 $$A^\mathrm{T}$$ 的行，对于上面的矩阵 $$A$$ 来说，我们有：

$$
A^\mathrm{T} =
\begin{bmatrix}
  1 & 3 & 5 \\
  2 & 4 & 6
\end{bmatrix}
$$

一个 $$N$$ 维向量也可以表示为一个矩阵的形式，也就是一个 $$N \times 1$$ 的矩阵：

$$
\vec{v} =
\begin{bmatrix}
  1 \\
  2 \\
  3
\end{bmatrix}
$$

为了减少版面占用，我们在写向量的时候往往写它们的转置形式：

$$
\vec{v} =
\begin{bmatrix}
  1 & 2 & 3
\end{bmatrix}^\mathrm{T}
$$

类似地，空间中的点也能用类似这一的表示。这里需要略微说明的是，由于坐标系中的一个点本身可以看作是一个从原点开始指向该点的向量，因此，在许多图形库中也常直接用向量来表示顶点。多数情况下，我们并不需要区分这两个概念，但是，在一些特定的场合（如后文将提到的平移变换）下，我们还是要严格区分点和向量的。

对于一个矩阵 $$A$$ 而言，我们用 $$A_{ij}$$ 表示这个矩阵第 $$i$$ 行第 $$j$$ 列的值。

矩阵之间可以进行加法和乘法运算，加法运算要求矩阵有相同的行数和列数，然后进行对应位置的相加：

$$
\begin{bmatrix}
  A_{11} & A_{12} & A_{13} \\
  A_{21} & A_{22} & A_{23}
\end{bmatrix}
+
\begin{bmatrix}
  B_{11} & B_{12} & B_{13} \\
  B_{21} & B_{22} & B_{23}
\end{bmatrix}
=
\begin{bmatrix}
  A_{11} + B_{11} & A_{12} + B_{12} & A_{13} + B_{13} \\
  A_{21} + B_{21} & A_{22} + B_{22} & A_{23} + B_{23}
\end{bmatrix}
$$

乘法运算则稍微复杂一点，对于 $$A$$、$$B$$ 两个矩阵相乘，我们需要确保 $$A$$ 的列数等于 $$B$$ 的行数。假设 $$A$$ 是一个 $$3 \times 2$$ 的矩阵，而 $$B$$ 是一个 $$2 \times 1$$ 的矩阵，则 $$A \ B$$ 的结果就是一个 $$3 \times 1$$ 的矩阵：

$$
\begin{bmatrix}
  A_{11} & A_{12} \\
  A_{21} & A_{22} \\
  A_{31} & A_{32}
\end{bmatrix}
\cdot
\begin{bmatrix}
  B_{11} \\
  B_{12}
\end{bmatrix}
=
\begin{bmatrix}
  A_{11} B_{11} + A_{12} B_{12} \\
  A_{21} B_{11} + A_{22} B_{12} \\
  A_{31} B_{11} + A_{32} B_{12}
\end{bmatrix}
$$

下面快速给出一组定义：

- 一个矩阵 $$A$$ 的主对角线（Main Diagonal）被定义为所有 $$A_{ij}, (i = j)$$ 的值。
- 对角矩阵（Diagonal Matrix）被定义为除了主对角线都为 $$0$$ 的矩阵。
- 方阵（Square Matrix）被定义为行列数相同的矩阵，一个 $$n \times n$$ 的方阵被称为 $$n$$ 阶方阵，$$n$$ 为方阵的阶。
- 单位矩阵（Identity Matrix）被定义为主对角线上的元素都为 $$1$$ 而其他元素都为 $$0$$ 的方阵，一个 $$n$$ 阶的单位矩阵记作 $$I_n$$。
- 给定一个 $$n$$ 阶方阵 $$A$$，如果存在一个 $$n$$ 阶方阵 $$B$$ 使得 $$A \ B = B \ A = I_n$$，则称 $$B$$ 是 $$A$$ 的逆矩阵（Inverse Matrix），记作 $$A^{-1}$$。
- 如果一个方阵 $$A$$ 的转置矩阵为其逆矩阵，即 $$A^\mathrm{T} = A^{-1}$$，则称 $$A$$ 为正交矩阵（Orthogonal Matrix）。

说完矩阵的一些相关的定义和运算之后，我们来说一下矩阵和我们的坐标变换有什么关系。由于整个坐标变换过程事实上是对模型的顶点应用了一组线性变换，因此它们可以被转变为用矩阵表示，例如：

$$
\begin{align}
  x^\prime &= x + y + z \\
  y^\prime &= 3x + 5z \\
  z^\prime &= 6x
\end{align}
$$

可以用矩阵乘法表示为：

$$
\begin{bmatrix}
  x^\prime & y^\prime & z^\prime
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  1 & 1 & 1 \\
  3 & 0 & 5 \\
  6 & 0 & 0
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y & z
\end{bmatrix}^\mathrm{T}
$$

这里不用简单的运算而引入矩阵这一概念，是因为矩阵乘法有一个很好的性质：运算服从结合律。因此，对一个点 $$p$$ 先应用矩阵 $$A$$ 再应用矩阵 $$B$$ 就相当于直接对 $$p$$ 应用 $$(A \ B)$$。在实际的场景中，我们可能需要处理非常大量的顶点，矩阵乘法的这一特性可以使得我们将变换过程计算一次，生成一个变换矩阵后，再将这个矩阵应用到所有顶点上，显著减少计算量。

# 图形基本变换

通过前文的介绍，我们已经了解了矩阵以及矩阵的一些基本运算方式。既然我们有了一个可组合的计算工具，现在我们就需要去了解我们可用的一些基础组合子，也就是图形变换的基本形式，以及这些变换如何用矩阵乘法的形式进行表示。在这里，以二维情况为例，说明图形几种基本的变换所对应的变换矩阵：

**缩放**

所谓缩放，其实就是对图形的每一个顶点的每一个分量都乘上一个缩放因子，例如我们想让一个二维图形在 $$x$$ 轴方向缩放 2 倍，在 $$y$$ 轴方向缩放 3 倍，那么，我们只需要对它上面的任何一个顶点 $$p = (x, y)^\mathrm{T}$$ 进行如下操作：

$$
\begin{align}
  x^\prime &= 2x \\
  y^\prime &= 3y
\end{align}
$$

将其写成矩阵形式：

$$
\begin{bmatrix}
  x^\prime & y^\prime
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  2 & 0 \\
  0 & 3
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y
\end{bmatrix}^\mathrm{T}
$$

也就是通用情况下，我们有：

$$
\begin{bmatrix}
  s_x & 0 \\
  0 & s_y
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  s_x x & s_y y
\end{bmatrix}^\mathrm{T}
$$

**旋转**

相比于缩放，旋转要稍微复杂一点。对于一个二维的图形的每一个顶点 $$p = (x, y)^\mathrm{T}$$ 我们希望对其应用一个 $$2 \times 2$$ 的矩阵，使得其绕原点逆时针旋转一个 $$\theta$$ 角后得到 $$(x^\prime, y^\prime)$$：

$$
\begin{bmatrix}
  a & b \\
  c & d
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  x^\prime & y^\prime
\end{bmatrix}^\mathrm{T}
$$

我们可以从两种最简单的情况进行考虑，来求出 $$a$$、$$b$$、$$c$$、$$d$$ 分别是什么。

第一种情况是 $$x \neq 0, \ y = 0$$。此时，矩阵相乘的结果为：

$$
\begin{align}
  \begin{bmatrix}
    x^\prime & y^\prime
  \end{bmatrix}^\mathrm{T}
  &=
  \begin{bmatrix}
    a & b \\
    c & d
  \end{bmatrix}
  \cdot
  \begin{bmatrix}
    x & 0
  \end{bmatrix}^\mathrm{T}
  \\
  &=
  \begin{bmatrix}
    a x + 0 & c x + 0
  \end{bmatrix}^\mathrm{T}
  \\
  &=
  \begin{bmatrix}
    a x & c x
  \end{bmatrix}^\mathrm{T}
\end{align}
$$

我们可以知道，对于这样的点而言，旋转 $$\theta$$ 角后，$$x^\prime = x\cos{\theta}$$，$$y^\prime = x\sin{\theta}$$。将这个结果对应于矩阵，我们就可以知道：

$$
\begin{align}
  a &= \cos{\theta} \\
  c &= \sin{\theta}
\end{align}
$$

类似地，对于 $$x = 0, \ y \neq 0$$ 的情况而言。此时，矩阵相乘的结果为：

$$
\begin{align}
  \begin{bmatrix}
    x^\prime & y^\prime
  \end{bmatrix}^\mathrm{T}
  &=
  \begin{bmatrix}
    a & b \\
    c & d
  \end{bmatrix}
  \cdot
  \begin{bmatrix}
    0 & y
  \end{bmatrix}^\mathrm{T}
  \\
  &=
  \begin{bmatrix}
    0 + b y & 0 + d y
  \end{bmatrix}^\mathrm{T}
  \\
  &=
  \begin{bmatrix}
    b y & d y
  \end{bmatrix}^\mathrm{T}
\end{align}
$$

对于这样的点而言，旋转 $$\theta$$ 角后，$$x^\prime = -y\sin{\theta}$$，$$y^\prime = y\cos{\theta}$$。将这个结果对应于矩阵，我们就可以知道：

$$
\begin{align}
  b &= -\sin{\theta} \\
  d &= \cos{\theta}
\end{align}
$$

因此，二维场景下，逆时针旋转的矩阵为：

$$
\begin{bmatrix}
  \cos{\theta} & -\sin{\theta} \\
  \sin{\theta} & \cos{\theta}
\end{bmatrix}
$$

**平移**

所谓平移，其实就是对点的每一个分量都加上一个偏移量，例如我们想让一个图形在 $$x$$ 轴方向平移 1 个单位长度，在 $$y$$ 轴方向平移 2 个单位长度，那么，我们只需要对其每一个顶点 $$p = (x, y)^\mathrm{T}$$ 进行如下操作：

$$
\begin{align}
  x^\prime &= x + 1 \\
  y^\prime &= y + 2
\end{align}
$$

这看起来比前面的两种变换都要简单，但事实上，我们却无法用上面的 $$2 \times 2$$ 矩阵乘以一个向量的方式表示这样的运算。这是因为对于任意的 $$2 \times 2$$ 矩阵而言，当其应用于一个向量的时候，它的结果总为如下形式：

$$
\begin{bmatrix}
  a & b \\
  c & d
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  a x + b y & c x + d y
\end{bmatrix}^\mathrm{T}
$$

我们会发现，这里并没有任何地方能放下一个与 $$x$$ 和 $$y$$ 都无关的值，因此，我们只能将其写为：

$$
\begin{bmatrix}
  a & b \\
  c & d
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y
\end{bmatrix}^\mathrm{T}
+
\begin{bmatrix}
  t_x & t_y
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  a x + b y + t_x & c x + d y + t_y
\end{bmatrix}^\mathrm{T}
$$

这看起来似乎没什么问题，但是它导致了一个巨大的灾难，就是我们无法将若干矩阵变换应用结合律合成一个矩阵了，这是我们不想看到的。因此我们需要引入一个新的数学工具：齐次坐标（Homogeneous Coordinates）。在齐次坐标下，一个点 $$p = (x, y)^\mathrm{T}$$ 将被表示为 $$p = (x, y, 1)^\mathrm{T}$$，向量 $$\vec{v} = (x, y)^\mathrm{T}$$ 将被表示为：$$\vec{v} = (x, y, 0)^\mathrm{T}$$。在引入其次坐标后，我们对点的平移就可以用如下的矩阵乘法实现了：

$$
\begin{bmatrix}
  1 & 0 & t_x \\
  0 & 1 & t_y \\
  0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y & 1
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  x + t_x & y + t_y & 1
\end{bmatrix}^\mathrm{T}
$$

我们可以看到，此时，这个向量的前两个值恰好就是我们要的结果。前面提到，我们大多数情况下并不严格区分点和向量，这里之所以需要严格区分，是因为向量具有平移不变性。当一个向量应用了上述变换后，则变为：

$$
\begin{bmatrix}
  1 & 0 & t_x \\
  0 & 1 & t_y \\
  0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
  x & y & 0
\end{bmatrix}^\mathrm{T}
=
\begin{bmatrix}
  x & y & 0
\end{bmatrix}^\mathrm{T}
$$

结果还是原来的向量，确保了平移不变性。

在使用了其次坐标后，本章提到的三个基本的变换矩阵可以分别被表示为：

- 缩放

  $$
  S(s_x, s_y) =
  \begin{bmatrix}
    s_x & 0 & 0 \\
    0 & s_y & 0 \\
    0 & 0 & 1
  \end{bmatrix}
  $$

- 旋转

  $$
  R(\theta) =
  \begin{bmatrix}
    \cos{\theta} & -\sin{\theta} & 0 \\
    \sin{\theta} & \cos{\theta} & 0 \\
    0 & 0 & 1
  \end{bmatrix}
  $$

- 平移

  $$
  T(t_x, t_y) =
  \begin{bmatrix}
    1 & 0 & t_x \\
    0 & 1 & t_y \\
    0 & 0 & 1
  \end{bmatrix}
  $$

**组合变换**

有了这些基础的变换方式，我们就可以组合成一些更复杂的变换形式。例如，我们刚刚提到的旋转变换是基于原点逆时针旋转 $$\theta$$ 角，那如果我们想绕任意一个点 $$p = (a, b)$$ 旋转 $$\theta$$ 角要怎么做呢？我们可以将这个过程拆解为三步：

- 将整个图形和点 $$p$$ 一起移动，使得点 $$p$$ 被移动到原点
- 将图形绕原点旋转
- 将整个图形点 $$p$$ 移动回原位置

也就是：

$$
R_p = T(a, b) \cdot R(\theta) \cdot T(-a, -b)
$$

**从二维空间到三维空间**

前文已经说明如何对二维情况下的点和向量进行变换，对于三维情况，我们也可以做类似的处理。我们首先通过齐次坐标将三维空间中的点 $$p = (x, y, z)^\mathrm{T}$$ 扩充为 $$p = (x, y, z, 1)$$，将三维空间中的向量 $$\vec{v} = (x, y, z)^\mathrm{T}$$ 扩充为 $$\vec{v} = (x, y, z, 0)$$。然后，对应的变换矩阵就变为：

- 缩放

  $$
  S(s_x, s_y, s_z) =
  \begin{bmatrix}
    s_x & 0 & 0 & 0 \\
    0 & s_y & 0 & 0 \\
    0 & 0 & s_z & 0 \\
    0 & 0 & 0 & 1
  \end{bmatrix}
  $$

- 平移

  $$
  T(t_x, t_y, t_z) =
  \begin{bmatrix}
    1 & 0 & 0 & t_x \\
    0 & 1 & 0 & t_y \\
    0 & 0 & 1 & t_z \\
    0 & 0 & 0 & 1
  \end{bmatrix}
  $$

- 旋转

  旋转比较特殊，在二维空间中，绕原点逆时针旋转含义是非常明确的，但在三维空间中，我们则无法说绕原点逆时针旋转，而是需要确定是绕哪个轴旋转，它们的公式分别为：

  - 绕 $$x$$ 轴旋转

    $$
    R_x(\theta) =
    \begin{bmatrix}
      1 & 0            & 0             & 0 \\
      0 & \cos{\theta} & -\sin{\theta} & 0 \\
      0 & \sin{\theta} & \cos{\theta}  & 0 \\
      0 & 0            & 0             & 1
    \end{bmatrix}
    $$

  - 绕 $$y$$ 轴旋转

    $$
    R_y(\theta) =
    \begin{bmatrix}
      \cos{\theta}  & 0 & \sin{\theta}  & 0 \\
      0             & 1 & 0             & 0 \\
      -\sin{\theta} & 0 & \cos{\theta}  & 0 \\
      0             & 0 & 0             & 1
    \end{bmatrix}
    $$

  - 绕 $$z$$ 轴旋转

    $$
    R_z(\theta) =
    \begin{bmatrix}
      \cos{\theta} & -\sin{\theta} & 0 & 0 \\
      \sin{\theta} & \cos{\theta}  & 0 & 0 \\
      0            & 0             & 1 & 0 \\
      0            & 0             & 0 & 1
    \end{bmatrix}
    $$

# 坐标变换

有了这些基础知识后，我们就可以来讨论完整的坐标变换应该怎么做了。文章开头已经提到，在空间中的顶点坐标变换分为三步。而这三步中的每一步都可以用一个对应的变换矩阵实现，它们是：

- 模型空间到世界空间：Model Matrix
- 世界空间到观察空间：View Matrix
- 观察空间到裁剪空间：Projection Matrix

根据这三个变换的首字母，这个过程又被称为 MVP 变换。

首先，Model Matrix 是比较简单的情况，它其实就是一些基础的变换或者是基础变换的组合，将物体的顶点从模型中定义的坐标系移动到世界坐标系中，例如一个正方体的盒子的一个顶点在 $$(1, 1, 1)$$ 的位置上，那么当这个正方体移动到了 $$(2, 3, 5)$$ 位置上时，这个顶点也自然应该被移动到 $$(3, 4, 6)$$ 位置上了。

而 View Matrix 概念上虽然是移动相机，但是本质上也和 Model Matrix 没有什么不同，也可以看作是为移动物体。如何理解这件事呢？首先我们来看相机的位置如何定义。当我们定义相机的位置时，一般会采用这样的方式：

- 位置 Position，用向量 $$\vec{e}$$ 表示
- 朝向 Look-at Direction，用单位向量 $$\vec{g}$$ 表示
- 上方 Up Direction，用单位向量 $$\vec{t}$$ 表示

我们知道，位置是相对的，假设我们正拿着一个相机在拍摄一个物体，固定好位置并拍出一张相片后，我们将相机和被拍摄物体都向前移动一段相同的距离，再向左移动相同的距离，然后再拍摄一张照片，在不考虑背景的情况下，这两张照片拍摄出来的结果显然是一模一样的。这也就意味着，我们可以根据计算的便利性，选择一个坐标系，来将所有物体和相机都按照这个坐标系进行移动。在这里我们选择就以相机的位置为原点，相机的上方向 $$\vec{t}$$ 为 y 轴的正方向，相机看向的方向 $$\vec{g}$$ 为 z 轴的负方向，以此为基础构建一个右手的坐标系（也就是 x 轴向右，y 轴向上的情况下，z 轴向屏幕外，$$\vec{x} \times \vec{y} = \vec{z}$$）。

也就是说，我们需要先对场景中所有物体，包括相机，应用一个变换矩阵，使得相机恰好在原点，相机的上方向 $$\vec{t}$$ 为 y 轴的正方向，相机看向的方向 $$\vec{g}$$ 为 z 轴的负方向。为了达成这个目的，我们先对场景中的物体应用一个平移变换，这个变换是非常直观的：

$$
T_{view} =
\begin{bmatrix}
  1 & 0 & 0 & e_x \\
  0 & 1 & 0 & e_y \\
  0 & 0 & 1 & e_z \\
  0 & 0 & 0 & 1
\end{bmatrix}
$$

在这个基础上，我们再应用旋转变换，使相机的 $$\vec{g}$$ 和 $$\vec{t}$$ 符合我们的需求。在这里我们需要应用一个小技巧，旋转矩阵是一个正交矩阵，根据前面给出的定义，一个矩阵




-----------------

这三个变换矩阵中比较特殊的是 Projection Matrix。在