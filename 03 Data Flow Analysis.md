# 一.引入

* Overview of Data Flow Analysis

* Preliminaries of Data Flow Analysis

* Reaching Definitions Analysis
* Live Variables Analysis

* Available Expressions Analysis

首先我们来解构一下什么是Data Flow Analysis：

<img src="Pic\3.01.jpg" style="zoom:75%;" />

* **Data**：表征抽象域中的元素值
* **Flow**：代表数据流动的方法论，可以是may analysis，也可以是must analysis
* **CFG**：实际上就是我们分析的基础结构，主要涉及对Node以及Edge的处理，其分别对应Transfer function与Control-flow handling。

> 不同的数据流分析应用具有不同的data abstraction以及flow safe-approximation策略，因此最终的transfer function和control-flow handling也就不同。

# 二.基础概念

## 1.Input&Output State

<img src="Pic\3.02.jpg" style="zoom:80%;" />

* Each execution of an IR statement transforms an `input state` to a new `output state`

* The input (output) state is associated with the `program point` before (after) the statement

## 2.DFA的目的

<img src="Pic\3.03.jpg" style="zoom:80%;" />

数据流分析的目的，实际上就是能够把所有program point都与一个data-flow value联系起来，后者的值域实际上就是表征程序所有可能被观察的状态的抽象域domain。

例子：

<img src="Pic\3.04.jpg" style="zoom:80%;" />

<img src="Pic\3.05.jpg" style="zoom:80%;" />

因此数据流分析实际上就是对所有的statements在safe-approximation directed的`约束`上求解Input与Output。

所谓的约束实际上就是transfer function与flows of control，也即承载数据流的基础结构。约束本质上既取决于程序的形态结构，也取决于你如何将这些结构描述为“约束”的方法论。

接下来我们将用更形式化的语言来描述这些约束与IN/OUT的关系。

## 3.细说Constraints

### （1）Transfer Function

首先是对于基础语句约束抽象（就是一个函数）：

<img src="Pic\3.06.jpg" style="zoom:80%;" />

其中分析的方向也是约束的一部分。

### (2)Control Flow

之后对于控制流约束的抽象相对比较复杂，我们可以将其分为`BB之内的流`与`BB之间的流`：

<img src="Pic\3.07.jpg" style="zoom:80%;" />

分析的方向同样也会导致不同的约束形态：

<img src="Pic\3.08.jpg" style="zoom:80%;" />

# 三.DFA的应用

接下来的例子中，数据流分析不会涉及Method Call，也不会涉及Aliases（指向同一片内存的变量别名）。

## 1.Reaching Definition Analysis

### （1）定义

<img src="Pic\3.09.jpg" style="zoom:80%;" />

* definition：表征分配给v一个值。
* reaches q：definition可达q，也即定义在q点依旧有效。
  * 存在一条由定义点到q点的路径
  * definition没有在路径上被其它定义覆盖。

> Reaching Definition是may-analysis，原因在于两个点间的路径是多条的，只要有一条路径满足我们的要求（定义可以到达q点），我们就会宣称定义可达q点。

Reaching Definition一般用在`检测未定义的变量`上：在CFG的Entry中为v定义一个dummy definition，看看其是否能够可达到p点，如果可达（may-analysis存在一条路径可达），就表明可能出现未初始化的变量的使用。

### （2）实现机制

**Abstraction：**

我们关心的状态量实际上就是程序中所有变量的definition，因此我们可以将其抽象为`bit vector`，作为供我们观察的流动的“数据”。

<img src="Pic\3.10.jpg" style="zoom:70%;" />

其中1表示相应definition能够到达这个程序点p，反之代表定义不能够到达这个程序点。

**safe-approximation：**

<img src="Pic\3.11.jpg" alt="3.11" style="zoom:80%;" />

针对transfer function与control flow这两种主要的程序结构主要组成部分，我们结合safe-approximation构建`具体的约束`：

<img src="Pic\3.12.jpg" alt="3.12" style="zoom:60% ;" align="left" />

就是说我们经过一个BB，这个BB里面包含了涉及若干个变量的definition，首先我们需要把IN[B]中所有涉及这些变量的definition全部kill掉，然后激活这些新建立的definition。

<img src="Pic\3.13.jpg" style="zoom:60%;" />

将meet operator实例化为求并运算。

### （3）算法

<img src="Pic\3.14.jpg" style="zoom:80%;" />

一般OUT[B]的初始化设为∅表征may-analysis，设为全1表征must-analysis。

> * 这个算法一定会收敛吗？
>
>   由于对于每个BB，gen和kill的元素都是固定的，这就导致OUT[B]只会增长而不会缩减。而definition集是有限的，所以算法一定会收敛。
>
> * 算法最终收敛到的点是safe的吗？
>
>   算法最终收敛到一个不动点，其和程序的单调性有关。

例子：

<img src="Pic\3.15.jpg" style="zoom:80%;" />

## 2.Live Variables Analysis

### （1）定义

<img src="Pic\3.16.jpg" style="zoom:80%;" />

所谓的活变量分析实际上就是判断变量v的某个值在程序点p是否是`活的`，其是活的定义为存在以p为起始点的路径，在这条路径上v会被使用，并且在使用之前没有定义覆盖掉其在p点的值。

活变量分析可以被应用在寄存器分配上，比如说某个时刻所有寄存器都满了，但是我们需要使用一个寄存器，此时优先替换出承载dead value的寄存器是比较合适的。

> Live Variables Analysis是may-analysis，这是因为只要v在一条路径上满足我们的要求（被使用），我们就可以宣称其在p点是live的。
>
> Q：为什么不能是must-analysis？safe标准的定义是什么？

### （2）实现机制

**Abstraction：**

关注程序中所有变量的存活状态，因此同样可以将流动的数据抽象为`bit vector`。

<img src="Pic\3.17.jpg" style="zoom:80%;" />

**safe-analysis：**

在这里我们一般使用`反向分析`。原因在于如果采用正向分析，对于每一条路径我们必须从头到尾遍历一遍，并且针对use或者redefine的信息还需要依次往回重传到p点；而使用反向分析，可以顺带着回传有用信息。

<img src="Pic\3.18.jpg" style="zoom:80%;" />

对于控制流的处理依旧很好理解，而对于transfer function，我们对于BB的语句是从后往前进行分析的，因此其语义就是如果对v的使用出现在定义之前，我们会判定其在IN[B]处是live的。

### （3）算法

<img src="Pic\3.19.jpg" style="zoom:80%;" />

## 3.Available Expression Analysis

### （1）定义

<img src="Pic\3.20.jpg" style="zoom:80%;" />

表达式*`x op y`*在p点是available的，即为（1）所有从Entry开始到p的路径都要执行此表达式，（2）并且在此表达式的最后一次执行之后，没有针对x或y的重定义。

利用这个信息，我们实际上可以判断在一个程序点处某个表达式的结果是否可以被直接利用（available）。

> Available Variable Analysis是must-analysis，要求p点之上所有可能的路径都使用到此表达式并且没有被新的变量定义覆盖，我们才能宣称此表达式在p点是available的。

### （2）实现机制

**Abstraction：**

<img src="Pic\3.21.jpg" style="zoom:80%;" />

**Control-Flow Analysis：**

<img src="Pic\3.22.jpg" style="zoom:80%;" />

* 对于Transfer Function，我们可以kill掉所有在等式左侧的变量所涉及的表达式，并且gen右侧的表达式。

* 对于Control Flow，我们直接根据定义将meet op实例化为交集运算。

  <img src="Pic\3.23.jpg" style="zoom:80%;" />

> 值得注意的是，上面例子中两条路径里表达式都是available的，其值并不一样，但是这不会影响我们分析的结果，我们所做的只是布尔判断，具体使用AEA来进行优化的方法可能会很不一样（比如将所有的表达式都用一个临时变量t还替换，最后使用的表达式值是所属路径上的available值）

### （3）算法

<img src="Pic\3.24.jpg" style="zoom:80%;" />

值得注意的是，所有目标结果OUT[B]全部初始化为全1，这是为了使得没有在BB中出现的表达式都默认为available。

<img src="Pic\3.25.jpg" style="zoom:80%;" />

# 四.总结

<img src="Pic\3.26.jpg" style="zoom:80%;" />

其中Boundary的赋值实际上就隐含了分析的方向。

基于`结构`与`数据框架`理解上面的三种DFA：

* Reaching Definition：

  数据：表征某个程序点各个Definition是否有效（Reach到）

  结构：merge方案采用并集，表征只要有一条支路能传下来就可以的May-Analysis情况

* Live Variables

  数据：
