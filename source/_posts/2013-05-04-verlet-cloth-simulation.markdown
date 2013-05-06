---
layout: post
title: 简单布料物理模拟：使用韦尔莱积分法
date: 2013-05-04 10:42
comments: true
categories: Tech
---

TL;DR: 简单的[布料物理模拟][jsfiddle]。

[jsfiddle]: http://jsfiddle.net/metaphysiks/7KeaR/

不久前在Codepen上看到一个JavaScript实现的[简单布料力学模拟][tearable-cloth]，激起了我对柔体动力学模拟的兴趣。所谓*柔体动力学*，既是对可形变的物体的模拟。游戏和CG动画中的常见的衣服、布料和人体皆可认为是可形变的物体。对这类物体的物理模拟有多种方法，其中最为常见的便是基于*韦尔莱积分法*（Verlet integration）的物理模型。由于计算简单、资源消耗少，韦尔莱积分更是受游戏物理引擎开发者青睐。下面依样画葫芦，介绍如何使用韦尔莱积分法实现简单的布料力学模拟。

[tearable-cloth]: http://codepen.io/stuffit/pen/KrAwx

<!-- more -->

* Table of Contents
{:toc}

## 数值积分

先从维基百科摘一段*数值积分*的定义：

> 在数值分析中，数值积分是计算定积分数值的方法和理论。在数学分析中，给定函数的定积分的计算不总是可行的。许多定积分不能用已知的积分公式得到精确值。数值积分是利用黎曼积分等数学定义，用数值逼近的方法近似计算给定的定积分值。

### 为什么不使用欧拉积分法？

为了实现物理模拟，我们需要计算出系统中的物体在空间中的位置。而为了计算出物体在空间中的位置，我们最容易想到的就是使用*运动方程*：

$$
\rm \overrightarrow{Pos_2} = \overrightarrow{Pos_1} + \overrightarrow{v} \cdot \Delta{t}
$$

这种方法被称为*[欧拉法][euler]*。每次更新物体位置时，我们就将速度向量乘以步长[^timestep]计算出位置变化，然后加到当前位置上得到新的位置。欧拉法的问题在于，它的误差与步长有关。除非选择很小的步长，我们的计算误差会越来越大：

![euler-method](http://upload.wikimedia.org/wikipedia/commons/thumb/a/ae/Euler_method.png/613px-Euler_method.png)

[euler]: http://zh.wikipedia.org/wiki/%E6%AC%A7%E6%8B%89%E6%96%B9%E6%B3%95

[^timestep]: 我们定义两次更新的时间间隔为一个步长。

### 韦尔莱积分法

2001年的时候，丹麦数学家兼程序员Thomas Jakobsen在GDC[^gdc]上发表了一篇文章《[Advanced Character Physics][acp]》。他在文中提出，游戏物理引擎的*精确度*不是重点，最重要的是*可信度*（believability）和*性能*。他在文中提出了的物理模拟算法基于一种自60年代以来便用于分子动力学建模的方法——韦尔莱积分法。Thomas Jakobsen将他的算法应用于《杀手：代号47》的物理引擎中，这是最早为游戏角色引入*[布娃娃系统][ragdoll]*（Ragdoll Physics）并将其发扬光大的游戏之一。

[^gdc]: Game Developer's Conference

[ragdoll]: http://en.wikipedia.org/wiki/Ragdoll_physics

韦尔莱积分法和欧拉法很像，最主要的区别在于*速度*的表示。使用欧拉法，我们通过记录每一个物体的速度来计算下一帧的位置；使用韦尔莱积分法，我们则是应用物体的*惯性*和*加速度*来计算下一帧位置。换句话说，惯性是物体在上一帧的位置变化：

$$
\rm \overrightarrow{Pos_2} = \overrightarrow{Pos_1} + (\overrightarrow{Pos_1} - \overrightarrow{Pos_0}) + \frac{1}{2} \overrightarrow{a} \cdot {\Delta t}^{2}
$$

你肯定也看出来了，韦尔莱积分法的误差不比欧拉法更低。那你可能要问，干嘛不用欧拉法就好了？原因是，这种方法能有效地降低运算量。在满足物体间的约束条件时，我们不需要算出其速度，只要能求解其约束条件就可以了。韦尔莱积分法的这一性质使得它非常适合用于建立布料物理模型。没明白怎么回事？请接着往下看。

## 布料物理

所谓布料物理模拟，其实主要是模拟点和它们间的约束——你可以想象成是布料的纹路。点受外力在空间移动——比如风、重力，以及点间的弹性约束。比如下面图中的这块布（帘子？）就是用许多点和它们间的约束（也就是连接点的线）构成的网格：

![curtain-static](http://i.minus.com/iboD4qiRFUNvfc.png)

[acp]: http://graphics.cs.cmu.edu/nsp/course/15-869/2006/papers/jakobsen.htm

### 点

首先我们需要让点受外力影响并计算其位置变化，为此我们需要做两件事：

1. 将外力换算成加速度；
2. 将加速度利用积分转换成位置变化。

第一点使用牛顿第二定理就可以了：

``` coffeescript
@accX += fX / @mass
@accY += fY / @mass
```

第二点根据韦尔莱积分法，我们将「惯性」带来的位置变化和「加速度」带来的位置变化叠加：

``` coffeescript
velX = @x - @lastX
velY = @y - @lastY
tsSquare = ts * ts
# calculate the next position
nextX = @x + velX + 0.5 * @accX * tsSquare
nextY = @y + velY + 0.5 * @accY * tsSquare
# reset
@lastX = @x
@lastY = @y
@x = nextX
@y = nextY
@accX = 0
@accY = 0
```

为了直观地理解这一过程，我们假设空间中有一静止小球，然后我们给它一个瞬时加速度：

![ball-1](http://i.minus.com/iMUSAYnkDExzi.png)

当我们选择的步长足够小时，可以近似认为小球的在步长时间内位置变化是$$\rm \frac{1}{2} \overrightarrow{a} \cdot {\Delta t}^{2}$$：

![ball-2](http://i.minus.com/ixNOwcv4rSnoZ.png)

由于加速度的作用是瞬时的，小球接下来将受惯性影响继续向前运动。我们用`当前位置 - 上次位置`来隐式表示这一速度向量，于是下一时间步长内小球的运动就是：

![ball-3](http://i.minus.com/iupOs9Epd4YAI.png)

考虑到真实世界中的动能损失，我们还需要给小球一点阻力，这可以通过给速度一个很小的衰减来实现：

``` coffeescript
# dampen velocity
velX *= 0.99
velY *= 0.99
```

### 约束

若只是模拟简单的粒子系统中自由运动的小球，到这里也就可以了。不过呢，我们还需要约束点的运动。但我们的「约束」不是简单粗暴地将脱离原位的点重置到原始位置上，否则就没法用外力影响布料了。我们采用的方法是两个点之间定义一个*静止距离*（Resting Distance）——使得两个点处于稳态的距离。每次使用韦尔莱积分法更新系统中点的位置时，总会有些点靠得太近或太远，因此我们通过*约束求解*来调整点的位置，使得两点间为静止距离。

![resting-distance](http://i.minus.com/ibkvlhFYy8OBFc.png)

求解约束的方法很简单：两个点各移动相同的距离，使得移动后它们的距离为静止距离：

``` coffeescript
# distance between the two
diffX = @p1.x - @p2.x
diffY = @p1.y - @p2.y
dist = Math.sqrt diffX * diffX + diffY * diffY
# the ratio of how far along the resting distance the actual distance is
difference = (@restingDist - dist) / dist
# push/pull objects
@p1.x += diffX * 0.5 * difference
@p1.y += diffY * 0.5 * difference
@p2.x -= diffX * 0.5 * difference
@p2.y -= diffY * 0.5 * difference
```

然而，当超过两个点两两相连时，求解其中一对点的约束将会使得其他点间的约束无法满足。当然了，我们可以建立一个方程组同时求解满足约束条件的所有点的位置，但仅仅几个约束条件就会极大增加计算复杂度。因此呢，我们采用的方法是*多趟*求解约束（Demo中默认为5趟），随着求解约束次数的增加，我们认为这些点的位置将会越来越接近稳态。

### 构建布料

现在我们可以开始构建布料了。我们的规则很简单：将每个点连接到其左侧和上侧的点上（如果有的话），构建出一个网格：

![fabric](http://i.minus.com/ibrYNo66aa4Bmr.png)

最上方的一排点被称为「固定点」，它们的位置是固定不变的。

### 交互

物理引擎最重要的功能之一是交互，只有通过交互，使用者才能最直观地感受到对「现实」的仿真。

为布料添加鼠标左键拖动功能很简单，只要将鼠标附近的点往鼠标移动的方向移动即可：

``` coffeescript
if dist < MOUSE_INFLUENCE
  p.lastX = p.x - (@mouse.x - @mouse.lastX) * MOUSE_INFLUENCE_SCALAR
  p.lastY = p.y - (@mouse.y - @mouse.lastY) * MOUSE_INFLUENCE_SCALAR
```

另一个更有趣的交互是模拟布料的撕扯效果。如果我们拖动布料「太用力」的话，布料就会被撕破。实现方法很简单，在求解约束前判断两个点之间的距离是否超过某个阈值，来决定是否「切断」它们间的连接（也就是约束）：

``` coffeescript
# if the distance is more than tear sensitivity, the cloth tears
if dist > @tearSensitivity then @p1.removeLink this
```

还有一个右键拖动切割布料：

``` coffeescript
# handle right-click
if dist < MOUSE_TEAR_SIZE then p.links = []
```

## 结果

{% jsfiddle 7KeaR result,js,html,css default 570px %}

## 扩展

几个可以改进的地方：

1. 使用更复杂的约束条件：许多布料物理模拟采用了结构约束、切变约束和弯曲约束来达到更加可信的效果，参见[这里][devil-in-blue]；
2. 增加维度（3D）：虽然目前空间位置是二维表示的，但是可以轻易拓展到更高维度；
3. 求解约束次数（精确度）：虽然减少步长（即提高帧率）可以让模拟变得更精确，但是运算的时间开销使我们很难稳定地保持很小的步长；可以增加求解约束的趟数来提高精确度，但是趟数过多也会降低帧率；总之，我们需要在求解趟数和帧率间取得平衡；

[devil-in-blue]: http://www.darwin3d.com/gamedev/articles/col0599.pdf

## 参考

1. [Mosegaards Cloth Simulation Coding Tutorial](http://cg.alexandra.dk/tag/cloth-simulation/)
2. [Simulate Tearable Cloth and Ragdolls With Simple Verlet Integration](http://gamedev.tutsplus.com/tutorials/implementation/simulate-fabric-and-ragdolls-with-simple-verlet-integration/)
3. [Curtain and How To Build a Fabric Simulator](http://bluethen.com/wordpress/index.php/processing-app/curtain/)
4. [Devil in the Blue Faceted Dress: 
Real-time Cloth Animation](http://www.darwin3d.com/gamedev/articles/col0599.pdf)

## 脚注
