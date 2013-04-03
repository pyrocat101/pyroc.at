---
layout: post
title: LogoScript，一个Logo方言与实现
date: 2012-05-19 23:00
comments: true
categories: Tech
---

花了几天时间，完成了一个玩具脚本语言的解释器。我把这个语言命名为LogoScript。顾名思义，Logo是古老的绘图教学语言，Smalltalk的前身，现在大多数人知道它的就是“海龟作图“；取名中有Script，是因为它与JavaScript有着极其相似的语法。语言的实现只有两千余行代码，大部分由[CoffeeScript][1]编写。

## Feature

下面是几个LogoScript脚本以及对应的输出图像。

### Spiral

```js
for (x = 1; x <= 150; x++) {
  fd(1.5 * x);
  rt(89);
}
```

![spiral](http://i.minus.com/iPPRhJoG4v37z.png)

### Hypotrochoid

```js
for (i = 0; i < 36; i++) {
  for (j = 0; j < 36; j++) {
    fd(16);
    rt(10);
  }
  rt(10);
}
```

![hypotrochoid](http://i.minus.com/iKH8vlJuxyeo4.png)

### Spinning Squares

```js
function hue(theta) {
  red = round(127.5 * (1 + sin(theta)));
  green = round(127.5 * (1 + sin(theta + 120)));
  blue = round(127.5 * (1 + sin(theta + 240)));
  return 'rgb(' + red + ',' + green + ',' + blue + ')';
}

function square(side) {
  bk(side / 2);
  rt(90);
  bk(side / 2);
  pd();
  for (i = 0; i < 4; i++) {
    fd(side);
    lt(90);
  }
  pu();
  fd(side / 2);
  lt(90);
  fd(side / 2);
}

function spinSquare(side) {
  if (side < 12) return;
  setpc(hue(1.4 * side));
  square(side);
  lt(4);
  spinSquare(side - 8);
}

clear('black');
pu();
rt(6);
spinSquare(340);
```

![spinning-squares](http://i.minus.com/ibiMgVVGngyFVT.png)

### Tree

```js
function treeColor(length) {
	return 2.5 * (100 - length);
}

angle = 50;

function tree(length) {
  if (length < 10) return;
  setpw(length / 9);
  setpc('rgb(255,' + round(treeColor(length)) + ',0)');
  fd(length);
  lt(angle / 2);
  tree(length * 0.65);
	rt(angle);
	tree(length * 0.85);
	lt(angle / 2);
	pu();
	bk(length);
	pd();
}

clear('black');
pu();
setxy(-94, -160);
rt(5);
pd();
tree(85);
```

![tree](http://i.minus.com/iQ7AWBMiUQiuG.png)

### Twisted Roses

```js
function getColor(theta) {
  return round(abs(255 * cos(theta)));
}

function width(angle, radius) {
  seth2(radius * sin(angle), radius * cos(angle));
  return 2 * (1.5 + sin(45 + 2 * geth()));
}

function spiral(angle, twist) {
  // Twisted Rose Curves
  radius = 180 * sin(4 * angle);
  angle = angle + 20 * sin(twist * angle);
  setpw(width(angle, radius));
  setpc('rgb(' + getColor(30 + 3 * angle) + ',0,255)');
  setxy(radius * sin(angle), radius * cos(angle));
  pd();
}

clear('black');
pu();
//for (twist = 0; twist < 24; twist++) {
for (angle = 0; angle < 360; angle++) {
  spiral(angle, 7 /* twist */);
}
//}
```

![twisted-7](http://i.minus.com/i7riH9Bu93AfQ.png)

修改twist的值可以得到不同的玫瑰曲线，比如为5和为6的时候分别如下：

![twisted-5](http://i.minus.com/ib0u1XleXD1JKt.png)

![twisted-6](http://i.minus.com/irHFXA8nYJm0h.png)

<!-- more -->

## Implementation

LogoScript是动态类型的语言，支持函数的定义、递归以及常见的流程控制结构。源代码首先被编译成字节码（bytecode）后，交给一个栈虚拟机执行（类似Python的方式）。这样的设计为优化留下了很大空间，虽然目前几乎没有做任何优化。

所谓基于栈的虚拟机，其运算操作数隐式存在于一个运算栈中。举一个非常简单的例子：`a = b + c`，编译成Logo的字节码如下：

    LDLOCAL 1 (b)
    LDLOCAL 2 (c)
    ADD
    STLOCAL 0 (a)

可以看到，LDLOCAL指令将局部变量b和c压栈，ADD的作用是取出两个栈顶元素相加，结果值放入栈顶。我们的语言并不支持数组、字典等数据类型，因此操作码（opcode）只有40余种。

为了将源文件转换为字节码，我使用peg.js生成了文法分析器。文法分析器的作用是扫描源码，生成[抽象语法树][2]（AST）。经过两趟树遍历，AST被翻译为bytecode。至于绘图的部分，我通过node-canvas调用cairo库绘图，性能相当好。

不得不说，如果不是CoffeeScript的简洁和优雅，最终的源代码行数绝对远不止两千余行。另外，JavaScript语言本身的函数式编程特性也使得代码更加精炼。

有人会问，为什么使用一个解释型语言实现另一个解释型语言？性能不会很差吗？原因是，我在编写这个玩具编译器之初，就是为了尝试通过JavaScript实现另一种语言。高级语言的诸多特性使得开发的时间大大缩短了。除此之外，对于一个简单的玩具绘图语言，我并不打算把性能作为重要的考虑因素，更何况现在的性能完全在我的可接受范围内（这完全归功于node.js那超快的V8引擎）。

现在的源代码托管在[github][3]上，附带了例子和简单的测试。

[1]: http://coffeescript.org
[2]: http://en.wikipedia.org/wiki/Abstract_syntax_tree
[3]: https://github.com/metaphysiks/LogoScript
