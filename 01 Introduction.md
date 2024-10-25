# **一**.PL与静态分析

<img src="C:\Users\84222\Desktop\Static Analysis\Pic\1.01.jpg" style="zoom:80%;" />

* Theory：关于程序设计语言的结构本身。比如语言的设计、类型系统、语义证明等等。
* Environment：支撑程序设计语言的环境系统。比如编译器、运行环境。
* Application：如何保证程序设计语言的效率、可靠性、安全性。

> **Background**: In the last decade, the language *cores* had few changes, but the programs became significantly larger and more complicated.
>
> **Challenge**: How to ensure the reliability, security and other promises of large-scale and complex programs?

# 二.静态分析

## 1.静态分析的定义

**静态分析**：在程序运行之前分析程序，得到程序在运行时的某些行为特征。

* 程序可靠性

  Null pointer dereference, memory leak

* 程序安全性

  Private information leak, injection attack

* 编译器优化

  Dead code elimination, code motion

* 程序理解

  IDE call hierarchy, type indication

## 2.莱斯定理

但是基于**Rice’s Theorem**，我们无法通过静态分析准确判断程序是否满足某些non-trivial properties：

![](C:\Users\84222\Desktop\Static Analysis\Pic\1.02.jpg)

## 3.Sound＆Complete

既然完美的静态分析是不存在的，所以我们只能在Sound与Complete之间进行权衡。

<img src="C:\Users\84222\Desktop\Static Analysis\Pic\1.03.jpg" style="zoom:80%;" />

* Compromise completeness: Overapproximate——不纯洁但是稳妥
* Compromise soundness: Underapproximate——纯洁但是不稳妥

大多数情况我们选择牺牲Completeness，从而得到较为稳妥的结果。

> 在静态分析的时候，我们要绝对保证Sound，在此基础上，我们还需要考虑是否值得消耗性能来获得更高的分析精度(从Sound外向内缩圈以获取精度是需要以牺牲性能为代价的)，也即进行precision和speed之间的权衡。

## 4.静态分析的过程

对于大部分静态分析来说，实际上就是**Abstraction + Over-approximation**。前者意味着我们需要抽象出一个结果域，这个域中所包含的元素是由静态分析问题的具体目的所决定的；后者实际上代表我们对实际域中的结果进行映射的方法论（同时也包含对于控制流的处理）。

下面以分析程序中所有变量的正负号为例进行简要说明（正负号分析可以检查除零错误以及数组负溢出错误）。

1. ==Abstraction==

<img src="C:\Users\84222\Desktop\Static Analysis\Pic\1.04.jpg" style="zoom:80%;" />

根据问题的目的，我们抽象出+、-、0三种基本的结果，另外的两种例外项是一般静态分析中为了保证抽象域覆盖逻辑全域所进行的必要抽象。

2. ==Over-approximation==

这种映射的方法论主要体现在转换函数Transfer Functions与控制流Control Flow的处理上。

* **转换函数Transfer Functions**

  转换函数定义如何将不同程序的语句映射为抽象域的值，其一般取决于原始程序语句的语义（具体域中的元素）以及所分析问题的导向（抽象域的结构）。

  <img src="C:\Users\84222\Desktop\Static Analysis\Pic\1.05.jpg" style="zoom:80%;" />

  比如在这个例子中，就是程序语句本意与设置的抽象域结构相结合，而且在转换中要注意Over-approximate的方法论。

* **控制流Control Flow**

  实际的程序并不是完全单向线性的，其会包含很多控制流/逻辑分支，一般我们直接将多个分支所流向的语句抽象为unknown。
  
  <img src="C:\Users\84222\Desktop\Static Analysis\Pic\1.06.jpg" style="zoom:80%;" />
  
  > 我们之所以如此粗鲁地这样抽象，是因为现实中的分支数量可能会很多，大部分情况很难全部枚举所有的可能性，这实际上就是precision & speed的权衡。
  >
  > 但是值得注意的是，我们有时也会选择性的分析几个分支（如果分支数量很少且简单，我们甚至可以全部枚举），这取决于实际情况。
