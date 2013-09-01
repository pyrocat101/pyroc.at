---
layout: post
title: Git的工作原理（一）：对象和引用
date: 2013-09-01 22:33
comments: true
categories: Essay
---

Git的用户界面简直[糟透了][git-koans]。它既不友好也不一致，不少命令和选项的取名也违反直觉。新手常常会不小心把工作目录和版本树折腾到一种不知如何复原的状态。然而剥去尚需花大力气改进的界面，它那强大的版本控制功能，都是建立在一个简单的核心之上的。本文中，我们将一窥其底层的工作原理。

[git-koans]: http://stevelosh.com/blog/2013/04/git-koans/

<!-- more -->

## 对象（Object）

*对象*是Git最核心的概念。Git将你的数据的所有版本存储在其*对象数据库*中；Git几乎所有的命令，都要和对象打交道。我们主要关心三种对象：blob、tree和commit。

### Blob

先做个实验，在一个空目录下：

``` bash
$ echo Hi > 1.txt
$ git init
$ git add .
$ find .git/objects -type f
```

你应该会看到这样的输出：`.git/objects/b1/4df6442ea5a1b382985a6549b85d435376c351`。

我是如何知道的呢？因为如下数据：

```
"blob" SP "3" NUL "Hi" LF
```

的SHA-1校验值是b14df6442ea5a1b382985a6549b85d435376c351。这里的`SP`是空格，`NUL`是空字符（ASCII码为0的字符），`LF`是换行符。

来验证一下：

``` bash
$ printf "blob 3\000Hi\n" | sha1sum
b14df6442ea5a1b382985a6549b85d435376c351  -
```

注意对象的名称和`1.txt`这个文件名没有关系，因为Git是*用内容寻址的*：文件不是根据其名称存储，而是根据其内容的校验值存储在blob对象中。既然每个对象都有独一无二的校验值，我们可以说用校验值寻址对象就是用内容来寻址对象。

因此我只要知道文件的内容，就可以计算出对象的路径：`.git/objects/<校验值前2位>/<校验值后38位>`。就算多个文件内容相同，Git也只会在`.git/objects`下存一份对象。

不过你无法直接查看这些对象的内容——它们都被压缩过了。不过你可以借助`zlib`过滤一下：

``` bash
$ alias zpipe="perl -MCompress::Zlib -e 'undef $/; print uncompress(<>)'"
$ cat .git/objects/b1/4df6442ea5a1b382985a6549b85d435376c351 | zpipe
blob 3Hi
```

也可以用Git提供的命令来打印出对象内容：

``` bash
$ git cat-file -p b14df6442ea5a1b382985a6549b85d435376c351
Hi
```

### Tree

既然blob对象不包含文件名信息，那么它们一定存放在某些别的地方，因为版本控制需要用文件系统的方式组织历史快照。

``` bash
$ git commit -m 'init'
$ find .git/objects -type f
```

这里会输出3个对象的路径，其中应该有一个是`.git/objects/d1/90eda3a45fd0d1682ff5bd94ece3cc5ab1ce25`。因为如下数据：

```
"tree" SP "33" NUL "10644 1.txt" NUL 0x b14df6442ea5a1b382985a6549b85d435376c351
```

的校验值是d190eda3a45fd0d1682ff5bd94ece3cc5ab1ce25。验证一下：

``` bash
$ git cat-file -p d190eda3a45fd0d1682ff5bd94ece3cc5ab1ce25
100644 blob b14df6442ea5a1b382985a6549b85d435376c351	1.txt
$ cat .git/objects/d1/90eda3a45fd0d1682ff5bd94ece3cc5ab1ce25 | zpipe | sha1sum
d190eda3a45fd0d1682ff5bd94ece3cc5ab1ce25  -
```

这样的对象就是tree。它是一个表，表中每一行有文件类型、文件名、对象类型和对象校验值这些信息。我们的例子中，100644表示一个`1.txt`是一个常规文件，它的内容由校验值为b14df6442ea5a1b382985a6549b85d435376c351的blob对象存储。当然了，每一行记录的也可以是其他的文件类型，比如符号链接和目录。如果是目录的话，那一行将是指向另一个tree对象的校验值。

### Commit

Commit对象包含了提交信息、作者、提交者、时间等信息。在我这里，刚才创建的commit对象位于`.git/objects/21/0ef855816fb85d12966dbacd640dab9dfca1ff`。

打印出来看看：

```
$ git cat-file -p 210ef855816fb85d12966dbacd640dab9dfca1ff
tree d190eda3a45fd0d1682ff5bd94ece3cc5ab1ce25
author pyrocat <i@pyroc.at> 1378036507 +0800
committer pyrocat <i@pyroc.at> 1378036507 +0800

init
```

每个commit都指向一个tree对象，由于这是首次提交，所以没有父commit对象的校验值。不过接下来的commit应该都会包含至少一行信息指出其父commit对象的校验值。

## 引用（Reference）

一个版本控制系统仅仅通过校验值标识对象是不够的。你当然用`git log 210ef85`这样的命令来查看版本历史，但是你每次都得记住最新的commit的校验值才行。有了*引用*，你才能用「人类」的方式操作Git。

引用其实就是一个指向对象的*指针*。Git中的不少概念本质上都是引用。

### 分支（Branch）

分支存放在`.git/refs/heads`下：

``` bash
$ find .git/refs/heads -type f
.git/refs/heads/master
```

每个分支是一个指向commit对象的引用：

``` bash
$ cat .git/refs/heads/master
210ef855816fb85d12966dbacd640dab9dfca1ff
```

它指向了首次创建的commit对象。下次在master分支上创建新的commit时，Git会为你更新`.git/refs/heads/master`为新commit的校验值。因此，每个分支本质上是一个会自动更新的引用。

### HEAD

表示当前分支的`HEAD`的路径是`.git/HEAD`，它是一个所谓的*符号引用*：

``` bash
$ cat .git/HEAD
ref: refs/heads/master
```

切换分支时，`HEAD`会更新：

``` bash
$ git checkout -b dev
$ cat .git/HEAD
ref: refs/heads/dev
```

## 参考

1. [Git Magic](http://www-cs-students.stanford.edu/~blynn/gitmagic/index.html)
2. [Git User's Manual](http://schacon.github.io/git/user-manual.html)
3. [Git from the bottom up](http://ftp.newartisans.com/pub/git.from.bottom.up.pdf)

未完待续。
