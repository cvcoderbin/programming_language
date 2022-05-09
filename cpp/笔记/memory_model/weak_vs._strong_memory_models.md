# Weak vs. Strong Memory Models

There are many types of memory reordering, and not all types of reordering occur equally often. It all depends on processor you’re targeting and/or the toolchain you’re using for development.

内存重新排序的类型有有多种，并不是所有类型的重新排序都同等频繁地发生。 这一切都取决于你的目标处理器和使用的开发工具链。

A **memory model** tells you, for a given processor or toolchain, exactly what types of memory reordering to expect at runtime relative to a given source code listing. Keep in mind that the effects of memory reordering can only be observed when [lock-free programming](http://preshing.com/20120612/an-introduction-to-lock-free-programming) techniques are used.

我们可以从内存模型获知，对于给定的处理器或工具链，相对于给定的源代码列表，在运行时期望什么样的内存重新排序。 请记住，只有在使用无锁编程技术时才能观察到内存重新排序的效果。

After studying memory models for a while – mostly by reading various online sources and verifying through experimentation – I’ve gone ahead and organized them into the following four categories. Below, each memory model makes all the guarantees of the ones to the left, plus some additional ones. I’ve drawn a clear line between weak memory models and strong ones, to capture the way most people appear to use these terms. Read on for my justification for doing so.

在研究了一段时间的内存模型之后——主要是通过阅读各种在线资源并通过实验进行验证——我将它们分为以下四类。 下面，每个内存模型都囊括了它左边那些的内存模型的所有保证，并增加了一些额外的保证。 根据大部分人使用这些术语的方式，我在它们之间划了一条明显的界限。继续往下读就能知道我为什么这么做了。

![img](F:\Git\programming_language\cpp\笔记\memory_model\Untitled.assets\weak-strong-table.png)



Each physical device pictured above represents a hardware memory model. A hardware memory model tells you what kind of memory ordering to expect at runtime relative to an *assembly* (or machine) code listing.

上图中的每个物理设备代表一个硬件内存模型。 硬件内存模型能告诉你对于汇编(或者机器)代码在运行期间会出现哪种内存执行顺序。

![img](F:\Git\programming_language\cpp\笔记\memory_model\Untitled.assets\hardware-matters.png)

Every processor family has different habits when it comes to memory reordering, and those habits can only be observed in multicore or multiprocessor configurations. Given that [multicore is now mainstream](http://preshing.com/20120208/a-look-back-at-single-threaded-cpu-performance), it’s worth having some familiarity with them.

每个处理器系列在内存重新排序方面都有不同的习惯，而这些习惯只能在多核或多处理器配置中观察到。 鉴于多核现在是主流，有必要对它们有所了解。

There are **software** memory models as well. Technically, once you’ve written (and debugged) portable lock-free code in C11, C++11 or Java, only the software memory model is supposed to matter. Nonetheless, a general understanding of hardware memory models may come in handy. It can help you explain unexpected behavior while debugging, and — perhaps just as importantly — appreciate how incorrect code may function correctly on a specific processor and toolchain out of luck.

还有软件内存模型。 从技术上讲，一旦你用 C11、C++11 或 Java 编写（并调试）了可移植的无锁代码，就应该只考虑软件内存模型。 尽管如此，对硬件内存模型的一般理解可能也会派上用场。 它可以帮助你在调试时解释意外行为，并且——也许同样重要的是——了解错误的代码如何在特定的处理器和工具链上正常运行。

## Weak Memory Models

[![img](F:\Git\programming_language\cpp\笔记\memory_model\Untitled.assets\analogy-small.png)](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations)In the weakest memory model, it’s possible to experience [all four types of memory reordering](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) I described using a source control analogy in a previous post. Any load or store operation can effectively be reordered with any other load or store operation, as long as it would never modify the behavior of a single, isolated thread. In reality, the reordering may be due to either [compiler reordering](http://preshing.com/20120625/memory-ordering-at-compile-time) of instructions, or memory reordering on the processor itself.

在最弱的内存模型中，有可能体验我在上一篇文章中使用源控制类比描述的所有四种类型的内存重新排序。 任何加载或存储操作都可以与任何其他加载或存储操作有效地重新排序，只要它永远不会修改单个隔离线程的行为。 实际上，重新排序可能是由于编译器对指令的重新排序或处理器本身的内存重新排序。

When a processor has a weak hardware memory model, we tend to say it’s *weakly-ordered* or that it has *weak ordering*. We may also say it has a *relaxed* memory model. The venerable **DEC Alpha** is everybody’s [favorite example](http://www.mjmwired.net/kernel/Documentation/memory-barriers.txt#2277) of a weakly-ordered processor. There’s really no mainstream processor with weaker ordering.

当处理器具有弱硬件内存模型时，我们倾向于说它是 *weakly-ordered*  的或 weak ordering 的。 我们也可以说它有一个宽松的内存模型。 令人敬仰的 DEC Alpha 是每个人都喜欢的弱序处理器的例子。现在主流的处理器中，没有比它更弱的内存模型了。

The C11 and C++11 programming languages expose a weak software memory model which was in many ways influenced by the Alpha. When using low-level atomic operations in these languages, it doesn’t matter if you’re actually targeting a strong processor family such as x86/64. As I demonstrated previously, you must still specify the [correct memory ordering constraints](http://preshing.com/20120913/acquire-and-release-semantics), if only to prevent compiler reordering.

受Alpha多方面的影响，C11与C++11编程语言呈现出一种弱软件内存模型。 如果你正在用x86/64等强序处理器系列，当在这些语言中使用底层的原子操作时并不会受到影响。 正如我之前所演示的，您仍然必须指定正确的内存排序约束，即使只是为了防止编译器重新排序。

### Weak With Data Dependency Ordering

Though the Alpha has become less relevant with time, we still have several modern CPU families which carry on in the same tradition of weak hardware ordering:

尽管 Alpha 随着时间的推移变得过时，但我们仍然有一些现代 CPU 系列仍然继承了弱硬件排序的传统：

- **ARM**, which is currently found in hundreds of millions of smartphones and tablets, and is increasingly popular in multicore configurations.

  ARM 目前已在数亿部智能手机和平板电脑中使用，并且在多核配置中越来越受欢迎。

- **PowerPC**, which the Xbox 360 in particular has already delivered to 70 million living rooms in a multicore configuration.

  PowerPC，特别是 Xbox 360 已经以多核配置向 7000 万客厅交

- **Itanium**, which Microsoft no longer supports in Windows, but which is still supported in Linux and found in HP servers.

  Itanium，Microsoft 在 Windows 中不再支持，但在 Linux 中仍受支持，并在 HP 服务器中找到。

These families have memory models which are, in various ways, almost as weak as the Alpha’s, except for one common detail of particular interest to programmers: they maintain [data dependency ordering](http://www.mjmwired.net/kernel/Documentation/memory-barriers.txt#305). What does that mean? It means that if you write `A->B` in C/C++, you are always guaranteed to load a value of `B` which is at least as new as the value of `A`. The Alpha doesn’t guarantee that. I won’t dwell on data dependency ordering too much here, except to mention that the [Linux RCU mechanism](http://lwn.net/Articles/262464/) relies on it heavily.

除了程序员特别感兴趣的一个常见细节之外，这些系列的内存模型在各种方面几乎和 Alpha 一样弱：它们维护数据依赖顺序。 这意味着什么？ 这意味着如果你在 C/C++ 中编写 A->B，你总是可以保证加载一个 B 的值，它至少与 A 的值一样新。 Alpha 不能保证这一点。 我不会在这里过多地讨论数据依赖排序，只 提一提Linux RCU 机制严重依赖它。

## Strong Memory Models

Let’s look at hardware memory models first. What, exactly, is the difference between a strong one and a weak one? There is actually [a little disagreement](http://herbsutter.com/2012/08/02/strong-and-weak-hardware-memory-models/#comment-5903) over this question, but my feeling is that in 80% of the cases, most people mean the same thing. Therefore, I’d like to propose the following definition:

我们先来看看硬件内存模型。 强内存模型与弱内存模型的区别是什么？ 这个问题其实有点分歧，但我的感觉是，在80%的情况下，大多数人的意思是一样的。 因此，我想提出以下定义：

> A **strong hardware memory model** is one in which every machine instruction comes implicitly with [acquire and release semantics](http://preshing.com/20120913/acquire-and-release-semantics). As a result, when one CPU core performs a sequence of writes, every other CPU core sees those values change in the same order that they were written.
>
> 强硬件内存模型是其中每条机器指令都隐含地带有 Acquire与Release 语义的模型。 因此，当一个 CPU 内核执行一系列写入时，其他每个 CPU 内核都会看到这些值的更改顺序与写入的顺序相同。

It’s not too hard to visualize. Just imagine a refinement of the [source control analogy](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) where all modifications are committed to shared memory in-order (no StoreStore reordering), pulled from shared memory in-order (no LoadLoad reordering), and instructions are always executed in-order (no LoadStore reordering). StoreLoad reordering, however, [still remains possible](http://preshing.com/20120515/memory-reordering-caught-in-the-act).

这些都不难理解。 想象一下源代码控制类比的改进，其中所有修改都按顺序提交到共享内存（没有 StoreStore 重新排序），按顺序从共享内存中提取（没有 LoadLoad 重新排序），并且指令总是按顺序执行（没有 LoadStore 重新排序）。 然而，StoreLoad 重新排序仍然是可能的。

![img](F:\Git\programming_language\cpp\笔记\memory_model\Untitled.assets\strong-hardware.png)

Under the above definition, the **x86/64** family of processors is *usually* strongly-ordered. There are certain cases in which some of x86/64’s [strong ordering guarantees are lost](http://preshing.com/20120913/acquire-and-release-semantics#comment-20810), but for the most part, as application programmers, we can ignore those cases. It’s true that a x86/64 processor can [execute instructions out-of-order](http://en.wikipedia.org/wiki/Out-of-order_execution), but that’s a hardware implementation detail – what matters is that it still keeps its *memory interactions* in-order, so in a multicore environment, we can still consider it strongly-ordered. Historically, there has also been a little confusion due to [evolving specs](http://jakob.engbloms.se/archives/1435).

在上述定义下，x86/64 系列处理器通常是 strongly-ordered 的。 在某些情况下，x86/64 的一些强排序保证会丢失，但在大多数情况下，作为应用程序员，我们可以忽略这些情况。 x86/64 处理器确实可以乱序执行指令，但这是一个硬件实现细节——重要的是它仍然保持其内存交互有序，所以在多核环境中，我们仍然可以它是 strongly-ordered的。 从历史上看，由于规范的不断发展，也存在一些混乱。

Apparently **SPARC** processors, when running in **TSO** mode, are another example of a strong hardware ordering. TSO stands for “total store order”, which in a subtle way, is different from the definition I gave above. It means that there is always a single, global order of writes to shared memory from all cores. The x86/64 has this property too: See Volume 3, §8.2.3.6-8 of [Intel’s x86/64 Architecture Specification](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) for some examples. From what I can tell, the TSO property isn’t usually of direct interest to low-level lock-free programmers, but it is a step towards sequential consistency.

显然，在 TSO 模式下运行时，SPARC 处理器是强硬件内存模型的另一个例子。 TSO 代表“total store order”，相比我上面给出的定义来看，这是一种微妙的方式。 这意味着从所有内核写入共享内存的顺序始终是单一的、全局的。 x86/64 也有这个属性：一些例子见英特尔 x86/64 架构规范的第 3 卷第 8.2.3.6-8 节。 据我所知，TSO性质不总是lock-free程序员对底层感兴趣的最直接部分，但这在对顺序一致性的理解上又迈进了一步。

### Sequential Consistency

In a [sequentially consistent](http://preshing.com/20120612/an-introduction-to-lock-free-programming#sequential-consistency) memory model, there is no memory reordering. It’s as if the entire program execution is reduced to a sequential interleaving of instructions from each thread. In particular, the result r1 = r2 = 0 from [Memory Reordering Caught in the Act](http://preshing.com/20120515/memory-reordering-caught-in-the-act) becomes impossible.

在顺序一致的内存模型中，没有内存重新排序。 就好像整个程序执行被简化为来自每个线程的指令的顺序交错。 特别是， r1 = r2 = 0  的结果在  [Memory Reordering Caught in the Act](http://preshing.com/20120515/memory-reordering-caught-in-the-act) 中变得不可能。

These days, you won’t easily find a modern multicore device which guarantees sequential consistency at the hardware level. However, it seems at least one sequentially consistent, dual-processor machine existed back in 1989: The 386-based [Compaq SystemPro](http://vogons.zetafleet.com/viewtopic.php?t=23842#178666). According to Intel’s docs, the 386 wasn’t advanced enough to perform any memory reordering at runtime.

如今你很难找到一个现代的多核设备能在硬件级别保证顺序一致性。然而，似乎在1989年至少有一种能保证顺序一致性的多处理器机器：基于386的[Compaq SystemPro](http://www.vogons.org/viewtopic.php?t=23842#178666)。根据Intel文档，386还不够先进到能在运行期间执行任意的内存乱序。

[![img](F:\Git\programming_language\cpp\笔记\memory_model\Untitled.assets\art-of-multiprocessor.png)](http://www.amazon.com/gp/product/0123973376/ref=as_li_ss_tl?ie=UTF8&tag=preshonprogr-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0123973376)In any case, sequential consistency only really becomes interesting as a **software** memory model, when working in higher-level programming languages. In Java 5 and higher, you can declare shared variables as `volatile`. In C++11, you can use the default ordering constraint, `memory_order_seq_cst`, when performing operations on atomic library types. If you do those things, the toolchain will restrict compiler reordering and emit CPU-specific instructions which act as the appropriate memory barrier types. In this way, a sequentially consistent memory model can be “emulated” even on weakly-ordered multicore devices. If you read Herlihy & Shavit’s [The Art of Multiprocessor Programming](http://www.amazon.com/gp/product/0123973376/ref=as_li_ss_tl?ie=UTF8&tag=preshonprogr-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=0123973376), be aware that most of their examples assume a sequentially consistent software memory model.

在任何情况下，只有在使用高级编程语言时，顺序一致性作为软件内存模型才真正变得有趣。 在 Java 5 及更高版本中，您可以将共享变量声明为 volatile。 在 C++11 中，您可以在对原子库类型执行操作时使用默认排序约束 memory_order_seq_cst。 如果你执行了这些操作，工具链将限制编译器重新排序并发出特定于 CPU 的指令，这些指令充当适当的内存屏障类型。 这样，即使在弱排序的多核设备上，也可以“模拟”顺序一致的内存模型。 如果您阅读 Herlihy & Shavit 的 The Art of Multiprocessor Programming，请注意他们的大多数示例都假设顺序一致的软件内存模型。

## Further Details

There are many other subtle details filling out the spectrum of memory models, but in my experience, they haven’t proved quite as interesting when writing lock-free code at the application level. There are things like control dependencies, causal consistency, and different memory types. Still, most discussions come back the four main categories I’ve outlined here.

要填完有关内存模型的坑，还有许多其它微妙的细节，但以我的经验来看，当在应用层上写无锁代码时，这些都变得没什么意思了。存在一些控制依赖，因果一致性和不同的内存乱序类型。然而，大部分的讨论都能归于我上面总结的四种主要类型里

If you really want to nitpick the fine details of processor memory models, and you enjoy eating formal logic for breakfast, you can check out the [admirably detailed work](http://www.cl.cam.ac.uk/~pes20/weakmemory/) done at the University of Cambridge. Paul McKenney has written an [accessible overview](http://lwn.net/Articles/470681/) of some of their work and its associated tools.

如果你真的想探究处理器内存模型的细节，并且你能在吃早餐的时候还想琢磨一些正式的逻辑，你可以查看剑桥大学所做的令人钦佩的详细工作。 Paul McKenney 撰写了一份易于理解的概述，介绍了他们的一些工作及其相关工具。

