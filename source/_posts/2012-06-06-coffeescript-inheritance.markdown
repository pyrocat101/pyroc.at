---
layout: post
title: CoffeeScript如何模拟类继承
date: 2012-06-06 23:31
comments: true
categories: Tech
---

CoffeeScript中可以使用`class`, `extends`和`super`等关键字模拟OOP中的类与继承，比如下面的CoffeeScript代码：

```coffeescript
class Animal
  constructor: (@name) ->

  move: (meters) ->
    alert @name + " moved #{meters}m."

class Snake extends Animal
  move: ->
    alert "Slithering..."
    super 5

class Horse extends Animal
  move: ->
    alert "Galloping..."
    super 45

sam = new Snake "Sammy the Python"
tom = new Horse "Tommy the Palomino"

sam.move()
tom.move()
```

将被翻译成：

```js
var Animal, Horse, Snake, sam, tom,
  __hasProp = {}.hasOwnProperty,
  __extends = function(child, parent) {
    for (var key in parent) {
        if (__hasProp.call(parent, key))
            child[key] = parent[key];
    }
    function ctor() {
        this.constructor = child;
    }
    ctor.prototype = parent.prototype;
    child.prototype = new ctor();
    child.__super__ = parent.prototype;
    return child;
  };

Animal = (function() {

  function Animal(name) {
    this.name = name;
  }
  Animal.prototype.move = function(meters) {
    return alert(this.name + (" moved " + meters + "m."));
  };

  return Animal;

})();

Snake = (function(_super) {

  __extends(Snake, _super);

  function Snake() {
    return Snake.__super__.constructor.apply(this, arguments);
  }

  Snake.prototype.move = function() {
    alert("Slithering...");
    return Snake.__super__.move.call(this, 5);
  };

  return Snake;

})(Animal);

Horse = (function(_super) {

  __extends(Horse, _super);

  function Horse() {
    return Horse.__super__.constructor.apply(this, arguments);
  }

  Horse.prototype.move = function() {
    alert("Galloping...");
    return Horse.__super__.move.call(this, 45);
  };

  return Horse;

})(Animal);

sam = new Snake("Sammy the Python");

tom = new Horse("Tommy the Palomino");

sam.move();

tom.move();
```

一开始看可能比较没头绪，所以下面将一步步渐进式地解释这种实现方式的含义。

<!-- more -->

先从Douglas Crockford给出的一个最简单的[原型继承函数][1]开始。输入：基对象`o`；输出：以`o`为原型的对象。

```js
extend = function (o) {
  var F = function () {};
  F.prototype = o;
  return new F();
}
```

注意JavaScript中`new`运算符设计得比较诡异，虽然试图模仿传统OOP，但是行为却不太一样。这里只需要知道，`new f()`所做的工作是创建一个对象，该对象继承自`f.prototype`。

好，接下来我们做点改进：传入的参数改为o的构造函数`B`。输入：基类构造函数`B`；输出：继承自`B`的对象。

```js
extend2 = function (B) {
  var F = function () {};
  var O = function () {};
  O.prototype = B.prototype;
  F.prototype = new O();
  return new F();
}
```

再进一步，我们需要让构造函数`F`不为空，考虑将我们的`extend`改为接受一个子类构造函数`F`和基类构造函数`B`，返回继承`B`的构造函数`F'`：

```js
extend3 = function (F, B) {
  var O = function () {};
  O.prototype = B.prototype;
  F.prototype = new O();
  return F;
};
```

使用：`Python = extend3(Python, Snake);`得到继承自Snake的Python。之后，可以加入Python类方法，例如：`Python.prototype.ear = function () { ... }`。

那么加入我们的基类构造函数也不是空函数呢？我们可能在构造函数`F`中先调用基类`B`的构造函数，因此需要提供访问基类构造函数的方法。例如`F`的定义可能是这个样子的：

```js
F = function () {
  this.__super__.constructor.apply(this, arguments);
};
```

那么，我们就需要在子类中提供`__super__`这个特殊成员用于访问基类`B`的构造器和方法。而且，提供`constructor`成员指向构造函数。

```js
extend4 = function (F, B) {
  // F.prototype.constructor == F
  var O = function () { this.constructor = F; }
  O.prototype = B.prototype;
  F.prototype.__super__ = B.prototype;
  F.prototype = new O();
  return F;
};
```

在C#和Java这样的面向对象语言中，类还有静态成员，比如C#中我们可以这样：

```csharp
Console.WriteLine(Math.PI);
Human.NumberOfLimbs = 4;
```

我们再对`extend`函数做点改进，使之也支持类静态成员（实际上就是构造函数的实例成员）：

```js
extend5 = function (F, B) {
  // copy base-class static members to sub-class
  for (var key in B) {
    if ({}.hasOwnProperty.call(B, key))
      F[key] = B[key];
  }
  var O = function () { this.constructor = F; }
  O.prototype = B.prototype;
  O.prototype.__super__ = B.prototype;
  F.prototype = new O();
  return F;
};
```

OK，这个函数改改变量名就基本和CoffeeScript中的实现继承的`extends`函数一致了。CoffeeScript中的这些语法糖就这样省去了自己手工编写和调用模拟类和继承的函数（这些很容易写错）。

[1]: http://javascript.crockford.com/prototypal.html
