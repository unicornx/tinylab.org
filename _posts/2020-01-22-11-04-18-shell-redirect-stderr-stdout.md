---
layout: post
author: 'Wang Chen'
title: "Shell 的内置（builtin）命令是什么，常常傻傻分不清"
draft: false
license: "cc-by-nc-nd-4.0"
permalink: /shell-redirect-stderr-stdout/
description: "Shell 中的 “2>&1” 命令，常常傻傻记不清，这次彻底搞清楚了"
category:
  - Shell
tags:
  - stderr
  - stdout
  - redirect
  - bash
---

> By unicornx of [TinyLab.org][1]
> Jan 22, 2020

在 Shell 编程中我们经常会在一些命令的尾巴上加上 “2>&1”，由于以前对它的语法一直是抱着得过且过的态度，所以一旦用起来就老是记不清楚，然后就是一通搜索、拷贝、黏贴。如此恶性循环，终于今天忍受不了自己对自己的放纵，上网搜了搜，搞得差不多清楚了，赶紧记下来和大家分享一下，希望真正成为自己的知识。


## 标准输出重定向

先从一个我们平时最常见的操作开始讲起。如果想看一个文件的内容，最常使用的是 cat 命令。默认情况下，cat 命令会将指定文件的内容打印出来并输出到一个接收输出的对象上，我们称这个接受程序打印输出内容的对象为 “标准输出（standard output）” ，简称 “stdout”。具体这个 stdout 是什么或指向谁，并不固定，默认情况下它指向我们计算机的屏幕，逻辑关系可以想象成这样：

```
command ---> stdout ---> screen
```

这里 command 在这里指代 `cat hello.txt`。换句话说，就是 cat 命令会将 hello.txt 的文件内容通过 stdout 输出显示在屏幕上，命令执行效果如下：

```
$ cat hello.txt
Hello world！
```

“Hello world!” 是 hello.txt 文件的内容，被 cat 打印在屏幕上了。

但是我们可以改变 stdout 指向的对象。这就涉及到 “重定向” 的概念了。首先我们要知道 Shell 的 “重定向” 功能分两种，一种输入重定向，一种是输出重定向；从字面上理解，输入输出重定向就是 “重新改变输入与输出的方向” 的意思。在这里我们主要以输出重定向为例，假设我们希望将 cat 命令的输出打印到一个名为 output.txt 的文件中去，而不是打印到屏幕上。其本质就是将 stdout 从原本指向屏幕改为指向 output.txt 这个文件，逻辑关系修改如下：

```
command ---> stdout ---> output.txt
```

为实现以上方式，命令行可以这么写：

```
$ cat hello.txt > output.txt

$ cat output.txt
Hello world！
```

其中 `>` 就是 shell 中实现 “重定向” 输出的操作符。观察第一条命令的执行结果我们会发现，cat 命令不会在屏幕上产生任何输出。因为我们已经将输出的默认位置更改为文件，所以 cat 命令会将 hello.txt 的内容打印到 output.txt 文件里，所以第二条 cat 命令在屏幕上输出 output.txt 的内容，我们看到了 hello.txt 的内容。


实际上我们常写的 `cat hello.txt > output.txt` 这种语法是简写形式，标准的写法如下：
```
$ cat hello.txt 1> output.txt
```

这里注意两点：
- `1` 是 stdout 在 Shell 中的代号（官方的说法是文件描述符的值，但本文不打算展开这个知识点），大部分用过 Linux/Unix 的人应该都是知道的。
- `1` 和 `>` 之间不可以有空格，这个是 Shell 的语法要求，`1> output.txt` 整体上表达的意思就是 stdout 现在指向了 output.txt。

## 标准出错重定向

如果我们执行 shell 的命令发生错误，譬如 cat 一个不存在的文件，如下所示，Shell 会产生出错信息。和正常的输出不同，Shell 会将出错信息输出给另一个特殊的对象，这个对象我们称之为 “标准出错（standard error）” ，简称 “stderr”。具体这个 stderr 是什么或指向谁，并不固定，和 stdout 一样，默认情况下它指向我们计算机的屏幕，逻辑关系可以想象成这样：

```
command ---> stderr ---> screen
```

具体命令执行的例子如下，假设 nop.txt 这个文件并不存在。

````
$ cat nop.txt
cat: nop.txt: No such file or directory
```

“cat: nop.txt: No such file or directory” 是 cat 命令执行出错后输出的内容，这里明显是打印在屏幕上了。

基于以上分析，我们应该可以自行分析以下命令为何还会在屏幕上看到出错的信息。

````
$ cat nop.txt > output.txt
cat: nop.txt: No such file or directory
```

出错信息仍然会显示在屏幕上。原因很简单，因为我们这里仅仅重定向了 “stdout”，并没有修改 “stderr” 的指向，所以出错信息当然还会出现在屏幕上。画一下上面这条命令对应的逻辑关系会更清楚：

```
command ---> stdout ---> output.txt
        |
        +--> stderr ---> screen
```

因此，如果你不想在屏幕上看到出错打印，可以采取的办法就是重定向 stderr，将其指向其他的设备，譬如一个文件。根据 stderr 在 Shell 中的代号是 `2`，并参考前面标准输出的完整写法，我们可以写出如下形式：

```
$ cat nop.txt 2> output.txt

$ cat output.txt
cat: nop.txt: No such file or directory
```

从运行结果中我们可以看到：出错信息被修改为打印到文件中，而不会显示在屏幕上了。逻辑关系图如下所示（注意我们这里补上了 stdout）：

```
command ---> stdout ---> screen
        |
        +--> stderr ---> output.txt
```

## 标准输出和标准出错同时重定向

那么最后问题来了，如果我们希望将正常的输出（标准输出）和出错信息（标准出错）都打印到 output.txt 文件里该怎么办?



```
command ---> stdout -+-> output.txt
        |            | 
        +--> stderr -+
```

## 文件描述符

文件描述符不过是代表打开文件的正整数。如果您有100个打开的文件，则将有100个文件描述符。

唯一需要注意的是，在Unix系统中，所有内容都是文件。但这现在并不重要，我们只需要知道标准输出（stdout）和标准错误（stderr）的文件描述符即可。

用简单的英语来说，这意味着存在标识这两个位置的“ id”，并且始终是1for stdout和2for stderr。

回到第一个示例，当我们将输出重定向cat foo.txt到时output.txt，我们可以像这样重写命令：

$ cat foo.txt 1> output.txt
这1只是的文件描述符stdout。重定向的语法是[FILE_DESCRIPTOR]>，省略文件描述符只是的快捷方式1>。


此时，您可能已经知道该2>&1成语在做什么，但让我们使其正式化。

您用于&1引用文件描述符1（stdout）的值。因此，当您使用时，您2>&1基本上是说“将重定向stderr到我们要重定向的同一位置stdout”。这就是为什么我们能够这样做既重定向stdout和stderr同一个地方：

$ cat foo.txt > output.txt 2>&1

$ cat output.txt
foo
bar
baz

$ cat nop.txt > output.txt 2>&1

$ cat output.txt
cat: nop.txt: No such file or directory

## 总结

程序将输出发送到两个位置：标准输出（stdout）和标准错误（stderr）。
您可以将这些输出重定向到其他位置（例如文件）。
文件描述符用于标识stdout（1）和stderr（2）；
command > output只是捷径command 1> output;
您可以&[FILE_DESCRIPTOR]用来引用文件描述符值；
使用2>&1将重定向stderr到设置为的任何值stdout（并且1>&2将执行相反的操作）。


如果您对这个主题感兴趣，请扫描二维码加微信联系我们：

![tinylab wechat](/images/wechat/tinylab.jpg)

[1]: http://tinylab.org
