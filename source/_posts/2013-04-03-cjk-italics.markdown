---
layout: post
title: 汉字中的「斜体」
date: 2013-04-03 18:00
comments: true
categories: Tech
---

在西文语境下，我们要在一句话中突出表示一个词，或者一句引语，按惯例往往采用*斜体*[^italic]。比如：I *love* you。随着计算机时代的到来，几乎所有文字处理软件，都内建了“斜体”这个功能。然而，崛起于西方世界的计算机科学，似乎从未考虑过东亚地区文字（CJK）的书写风格和历史习惯。在我们的的书写历史上，从未正式出现过使用意大利体来强调某个词语或者某句引语。由于汉字中并没有斜体，我们的文字处理软件中的“斜体”汉字，都是对正体字作倾斜变换后得到的*伪斜体*[^oblique]。

作为一个广为人知的误解，人们常常认为「斜体」就是「基准线向右倾斜的字体」，其实不然[^misuse]。「斜体」特指为了加快书写速度而字母略微向右倾斜的一种手写书法风格，而「基准线向右倾斜的字体」，即上面提到的「伪斜体」，仅仅是对*罗马体*西文字母做向右倾斜变换。

Donald Knuth在他的著作「$$\rm The\ \TeX book$$」中写道：

> Slanted type is essentially the same as roman, but the letters are slightly
> skewed, while the letters in italic type are drawn in a **different** style.
> (p. 13) [My emphasis]
{:lang='en'}

那么根据这个定义，中文字体中是否有对应于西文斜体的风格？在我看来，答案是肯定的。我们定义一个西文字体是不是斜体，最主要看是不是有手写风格在里面。换句话说，我们要寻找的是一种用不同风格书写的书法字体。在中文语境中，一个最为广泛使用，且满足上述条件的候选字就是：*楷体*。

<!-- more -->

现代浏览器对于`<em>`标签的默认样式仍然使用西文斜体，这显然没有考虑到东亚地区的文化传统。由于汉字字体没有斜体风格，浏览器对汉字做倾斜变换。这种汉字“伪斜体”即没有意义，也不具有任何观赏性可言。借助CSS，我们可以在中文语境下使用楷体表强调，而在西文语境下仍然使用斜体：

```css
@font-face {
  font-family: "Emphasis";
  src: local("HoeflerText-Italic"),
       local("Georgia Italic");
  font-style: normal;
  font-weight: normal;
}
```
上述片段使用`@font-face`指定访问者本地的Hoefler Text Italic或备用方案Georgia
Italic（Web安全字体）为Emphasis罗马体，这样我们在其他CSS规则的`font-family`中引用Emphasis这个名称，将斜体当做罗马正体来使用。

为`<em>`标签指定CSS规则如下：

```css
em {
  font-family: "Emphasis", STKaiti, Kai, Kaiti, serif;
  font-style: normal;
}
```

效果：

![CJK/Western](http://i.minus.com/ibdEYz1iVJvhZS.png)

## 参考

1. [CJK characters](http://en.wikipedia.org/wiki/CJK_characters)
2. [Roman与Italic的故事](http://www.typeisbeautiful.com/2010/01/1923/)
3. [What are Italics?](http://gniw.ca/cjk-italics.php)

## 注释

[^italic]: 即italic体，准确的翻译应该是“意大利体”。
[^misuse]: “斜体”不一定是斜的：[关于Italic的补充](http://www.typeisbeautiful.com/2010/03/2208/)
[^oblique]: 即slanted体或oblique体。

