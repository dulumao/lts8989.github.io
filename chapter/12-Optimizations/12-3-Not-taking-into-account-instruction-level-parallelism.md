## 12.3 CPU 指令级并行性对程序的优化

指令级并行性是另一个可能对性能产生重大影响的因素。在定义这个概念之前，让我们深入研究一个具体的例子，了解如何优化它。

我们将编写一个接收两个 `int64` 元素数组的函数。此函数将迭代一定次数（一个常数），并且在每次迭代期间，它将：

* 递增数组的第一个元素
* 如果第一个元素是偶数，则增加数组的第二个元素

这是 Go 版本：

```go
const n = 1_000_000

func add(s [2]int64) [2]int64 {
    for i := 0; i < n; i++ {
        s[0]++
        if s[0]%2 == 0 {
            s[1]++
        }
    }
    return s
}
```

在循环中执行的指令如下（让我们记住，增量需要读取和写入）：

![](https://img.exciting.net.cn/90.png)

顺序是连续的。首先，我们增加 `s[0]`，然后在增加 `s[1]` 之前，我们需要再次读取 `s[0]`。

> **Note** 该指令序列与汇编指令的粒度不匹配。然而，为了在本节中清楚起见，我们采用简化的视图。

让我们花点时间讨论指令级并行 (ILP) 概念的理论。几十年前，CPU 设计人员不再仅仅关注时钟速度来提高 CPU 性能。他们带来了多项优化，其中之一就是 ILP 的概念，它允许并行执行一系列指令。在单个虚拟内核中实现 ILP 的处理器称为超标量处理器。 

例如，下图表示一个 CPU 正在执行一个由三个指令 `I1`、`I2` 和 `I3` 组成的应用程序：

![](https://img.exciting.net.cn/91.png)

执行一系列指令需要不同的阶段。简而言之，CPU 需要对指令进行解码并执行它们。执行是由将执行不同操作和计算的执行单元处理。

在这种情况下，CPU 决定并行执行这三个指令。请注意，并非所有指令都必须在单个时钟周期内完成。例如，读取寄存器中已经存在的值的指令可以；然而，一条读取必须从主存储器中取出的地址的指令可能需要几十个时钟周期才能完成。

按顺序执行，该指令序列将花费以下时间（函数 `t(x)` 表示 CPU 执行指令 `x` 所花费的时间）：

```go
total time = t(I1) + t(I2) + t(I3)
```

多亏了 ILP，总时间现在如下：

```go
total time = max(t(I1), t(I2), t(I3))
```

ILP 理论上看起来很神奇。但是，它会导致一些称为危害的挑战。

例如，如果 `I3` 将变量设置为 42，但 `I2` 是条件指令（例如，`if foo == 1`）怎么办？ 理论上，它应该防止并行执行 `I2` 和`I3`。这称为控制风险（或分支风险）。在实践中，CPU 设计者使用分支预测解决了控制风险。

例如，CPU 可以计算出最近 100 次中有 99 次条件为真；因此，它将并行执行 `I2` 和 `I3`。如果预测错误（`I2` 恰好为假），CPU 将刷新其当前执行管道，确保不会导致任何不一致。这种刷新导致大约 10 到 20 个时钟周期的性能损失。

存在其他类型的危险，阻止并行执行指令。作为 Go 开发人员，我们可以产生影响。让我们考虑以下两条将更新寄存器（用于执行操作的临时存储区域）的指令：

* `I1` 将寄存器 A 和 B 中的数字加到 C
* `I2` 将寄存器 C 和 D 中的数字加到 D

因为 `I2` 取决于 `I1` 关于寄存器 C 值的结果，所以两条指令不能同时执行。`I1` 必须在 `I2` 之前完成。这是另一种称为数据冒险的冒险。

上图中，`I1`、`I2`、`I3` 被 CPU 并行执行。因此，不存在数据危害：

![](https://img.exciting.net.cn/92.png)

> **Note** 关于危害的几点注意事项：
> * 存在另一种类型的危险，称为结构性危险。然而，作为 Go 开发者，我们几乎无法影响它；因此我们将在本节中省略它。
> * 即使对于数据危害，CPU 设计人员也设想了一种称为转发的技巧，它基本上绕过了对寄存器的写入。它的目的不是解决这个问题，而是减轻影响。

现在，我们应该对 ILP 理论有一个不错的理解。让我们回到我们最初的问题，关注循环的内容：

```go
s[0]++
if s[0]%2 == 0 {
    s[1]++
}
```

正如我们所讨论的，数据冒险会阻止指令同时执行。让我们深入研究指令序列，但这一次，我们将强调指令之间的危险：

![](https://img.exciting.net.cn/93.png)

由于 `if` 语句，此序列包含一个控制风险。然而，正如所讨论的，优化它并预测应该采用哪个分支是 CPU 的范围。此外，我们可以注意到多个数据危害。正如我们所讨论的，数据冒险阻止 ILP 并行执行指令。从 ILP 的角度来看，以下是指令序列：

![](https://img.exciting.net.cn/94.png)

从 ILP 的角度来看，唯一独立的指令是 `s[0]` 检查和 `s[1]` 增量。因此，由于分支预测，这两个指令集可以并行执行。

增量呢？我们可以改进我们的版本以最小化数据危害的数量吗？

让我们编写另一个引入临时变量的版本（`add2`）：

```go
func add(s [2]int64) [2]int64 {
    for i := 0; i < n; i++ {
        s[0]++
        if s[0]%2 == 0 {
            s[1]++
        }
    }
    return s
}

func add2(s [2]int64) [2]int64 {
    for i := 0; i < n; i++ {
        v := s[0]
        s[0] = v + 1
        if v%2 != 0 {
            s[1]++
        }
    }
    return s
}
```

在这个新版本中，我们将 `s[0]` 的值固定为一个新变量 `v`。之前我们递增 `s[0]` 并检查它是否偶数。为了复制这种行为，由于 `v` 在递增之前基于 `s[0]`，我们现在检查 `v` 是否为奇数。 

现在让我们比较两个版本的危害：

![](https://img.exciting.net.cn/95.png)

我们可以注意到相同数量的步骤。显着差异在于数据风险。事实上，`s[0]` 增量步骤和检查 `v` 步骤现在依赖于相同的指令（将 `s[0]` 读入 `v`）。

为什么这有关系？因为它会让 CPU 提高并行度：

![](https://img.exciting.net.cn/96.png)

尽管步数相同，但第二个版本增加了可以并行执行的步数：三个并行路径而不是两个。同时，由于最长路径已减少，因此应优化执行时间。

如果我们对这两个函数进行基准测试，我们会注意到第二个版本有显着的加速提升（在我的机器上大约提高了 20%），主要是因为 ILP。

让我们回到本节的结论。我们讨论了现代 CPU 如何利用并行性来优化一组指令的执行时间。我们还讨论了一种阻止并行执行指令的主要风险类型：数据风险。此外，我们通过减少数据危险的数量来优化一个具体的 Go 示例，以增加可以并行执行的指令数量。

深入研究 Go 如何将我们的代码编译成程序集并了解如何利用 CPU 优化（如 ILP）可能是另一种改进途径。在这里，引入一个临时变量可以显着提高性能。这个例子展示了机械同情如何帮助我们优化 Go 应用程序。

让我们还记得对这种微优化保持谨慎。事实上，随着 Go 编译器在各个版本中不断发展，生成的应用程序程序集也会随着 Go 版本的变化而发生变化。

以下部分将讨论数据对齐的影响。