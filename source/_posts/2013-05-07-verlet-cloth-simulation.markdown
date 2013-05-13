---
layout: post
title: 简单布料物理模拟
date: 2013-05-07 18:52
comments: true
categories: Tech
---

太长不看：[Demo][jsfiddle]

[jsfiddle]: http://jsfiddle.net/metaphysiks/7KeaR/

不久前在Codepen上看到一个JavaScript实现的[简单布料力学模拟][tearable-cloth]，激起了我对柔体动力学模拟的兴趣。所谓*柔体动力学*，既是对可形变的物体的模拟。游戏和CG动画中的常见的衣服、布料和人体皆可认为是可形变的物体。对这类物体的物理模拟有多种方法，其中最为常见的便是基于*韦尔莱积分法*（Verlet integration）的物理模型。由于计算简单、资源消耗少，韦尔莱积分更是受游戏物理引擎开发者青睐。下面依样画葫芦，实现如何实现一个简单的布料物理模拟。

[tearable-cloth]: http://codepen.io/stuffit/pen/KrAwx

<!-- more -->

* Table of Contents
{:toc}

## 粒子

我们的物理系统的基本构件是*粒子*。一个粒子是一个空间中带有质量的点。

实现粒子物理最基本的想法是对它施加外力，计算它在空间中的位置变化，为此我们要做两件事：

1. 将外力换算成加速度；
2. 将加速度利用积分转换成位置变化。

第一点使用牛顿第二定理就可以了：

$$
\overrightarrow{a} = \frac{\overrightarrow{F}}{m}
$$

第二点则是把*惯性*带来的位置变化和*加速度*带来的位置变化叠加：

$$
\newcommand{\pos}[1]{\overrightarrow{P_{#1}}}
\pos{t+\Delta{t}} = \pos{t} + (\pos{t} - \pos{t-\Delta{t}}) + \frac{1}{2} \overrightarrow{a_t} {\Delta{t}}^2
$$

为了直观地理解这一过程，我们假设这样一个场景：空间中有一静止小球，给它施加一个外力。

![ball-1](http://i.minus.com/iMUSAYnkDExzi.png)

根据上面的公式，小球的上一时刻位置和当前时刻位置重合，「惯性」不影响小球位置，因此小球在$$\Delta{t}$$内的位置变化来自加速度：

![ball-2](http://i.minus.com/ixNOwcv4rSnoZ.png)

现在撤掉外力带来的加速度，小球只受惯性影响继续运动。当前位置和上次位置的差即是「惯性」导致的位置变化：

![ball-3](http://i.minus.com/iupOs9Epd4YAI.png)

由于加速度和惯性的影响是独立的，若小球同时具有惯性和加速度，只需要分别计算并叠加它们对位置的影响即可。

从加速度和时间计算出位置的过程称为积分。而上面介绍的这种积分方法就是*韦尔莱积分法*。那么你可能会问，为什么不用*欧拉法*呢？也就是用最直观的运动方程来积分：

$$
\pos{t+\Delta{t}} = \pos{t} + \overrightarrow{v_t} \Delta{t}
$$

原因在于，构成布料的粒子彼此间互相牵制，因此为了求解$$\overrightarrow{v_t}$$需要进行运算复杂度很高的受力分析。后面会看到，采用一种运算复杂度较低的近似算法，我们也可以达到媲美欧拉法的效果。

## 约束

*约束*是系统的规则，说白了就是规定粒子能在哪和不能在哪。我们要让粒子不能飞出边界；让粒子组成布料而不是一盘散沙；让一些粒子被「钉住」以模拟布料的悬挂。物理引擎正是通过满足约束为使用者带来「真实」的交互体验。

### 边界约束

为了不让布料飞出边界，我们要给组成布料的粒子一个简单的边界约束：

``` coffeescript
if x < 1 then x = 2 - x
if x > WIDTH - 1 then x = 2 * (WIDTH - 1) - x
```

同理，y轴方向上也需要应用相似的约束规则。

### 固定约束

有时候，我们希望系统中某些粒子的位置固定不变，这被称为「固定约束」。实现方法简单粗暴：

``` coffeescript
if pinned
  x = pinX
  y = pinY
```

### 结构约束

直观来说，结构约束是把一个粒子连到另一个粒子上的*链接*。为了保持布料的结构，我们希望约束粒子间的距离，也就是链接的长度为一个定值。这个定值被称为*静止距离*（Resting Distance）。每一帧更新所有粒子的位置时，粒子移动，可能使得有些链接的长度小于或大于静止距离，因此我们在*求解约束*的过程中要设法「矫正」它们的位置。方法很简单，让两个粒子各移动一半路程，使它们移动后的距离为静止距离：

![resting-distance](http://i.minus.com/ibkvlhFYy8OBFc.png)

不过这种简单粗暴、目光短浅的方法会带来一个问题：若超过两个以上的粒子互相链接，局部地满足其中一条链接的约束会破坏其他链接的约束，如[图][problem-1]所示。

[problem-1]: http://gamedevtuts.s3.amazonaws.com/022_verlet/link_constraint_faults.png

最好的解决办法是，列一个方程组，求解出同时满足所有约束条件的粒子位置，但其运算开销对于实时演算来说未免太大。因此我们采用的方法是：循环多次，每次循环里按顺序求解每个链接约束各一次。这个近似算法建立在一个观察的基础之上：随着求解约束的循环次数增加，粒子的位置越来越接近同时满足结构约束的状态。

## 布料建模

从粒子建立布料的规则很简单：将每个粒子分别链接到其左侧和上测的粒子上（如果有的话），构建出一个网格：

![fabric](http://i.minus.com/ibrYNo66aa4Bmr.png)

最上方的一排粒子被称为「固定点」，它们受固定约束限制，位置不因外部影响而改变。

## 交互

除了重力以外，布料收到的外部影响还有鼠标交互。

我们希望能用鼠标左键拖动布料。这只要让鼠标附近的粒子沿鼠标移动的方向移动即可：

``` coffeescript
if distance < MOUSE_INFLUENCE_SIZE
  p.lastX = p.x - (mouse.x - mouse.lastX) * MOUSE_INFLUENCE_SCALAR
  p.lastY = p.y - (mouse.y - mouse.lastY) * MOUSE_INFLUENCE_SCALAR
```

另一个有趣的交互是撕扯效果。如果鼠标拖动布料「太用力」的话，就会把布料撕破。实现方法也很简单，在求解约束前判断两个粒子之间的距离是否超过某个阈值，若超过了就「切断」它们间的链接：

``` coffeescript
# if the distance is more than tear sensitivity, the cloth tears
if distance > TEAR_SENSITIVITY then p1.removeLink link
```

还有一个右键拖动切割布料：

``` coffeescript
# clear links attached to the point
if distance < MOUSE_TEAR_SIZE then p.links = []
```

## 结果

*P.S. 如果觉得卡，请降低`CONSTRAINT_ACCURACY`的值。*{:lang='zh'}

{% codepen CkEmG metaphysiks result 570 %}

## 参考

1. [Mosegaards Cloth Simulation Coding Tutorial](http://cg.alexandra.dk/tag/cloth-simulation/)
2. [Simulate Tearable Cloth and Ragdolls With Simple Verlet Integration](http://gamedev.tutsplus.com/tutorials/implementation/simulate-fabric-and-ragdolls-with-simple-verlet-integration/)
3. [Curtain and How To Build a Fabric Simulator](http://bluethen.com/wordpress/index.php/processing-app/curtain/)
4. [Devil in the Blue Faceted Dress: 
Real-time Cloth Animation](http://www.darwin3d.com/gamedev/articles/col0599.pdf)
