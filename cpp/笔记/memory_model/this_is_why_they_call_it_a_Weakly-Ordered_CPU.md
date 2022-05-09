# This Is Why They Call It a Weakly-Ordered CPU

On this blog, I’ve been rambling on about [lock-free programming](http://preshing.com/20120612/an-introduction-to-lock-free-programming) subjects such as [acquire and release semantics](http://preshing.com/20120913/acquire-and-release-semantics) and [weakly-ordered CPUs](http://preshing.com/20120930/weak-vs-strong-memory-models). I’ve tried to make these subjects approachable and understandable, but at the end of the day, talk is cheap! Nothing drives the point home better than a concrete example.

在这一系列博客中，我已经详细讨论了[lock-free编程](https://chonghw.github.io/blog/2016/12/08/weakly-ordered-cpu/)相关的一系列主题，比如[Acquire与Release语义](http://www.chongh.wiki/blog/2016/09/28/acquireandrelease/)以及[Weakly-ordered CPUs](http://www.chongh.wiki/blog/2016/10/30/memorymodel/). 我已经努力尝试把这些主题讲的更加地通俗易懂，但大家都知道，Talk is cheap.没有言语比具体的例子更能把问题说清楚。

If there’s one thing that characterizes a weakly-ordered CPU, it’s that one CPU core can see values change in shared memory in a different order than another core wrote them. That’s what I’d like to demonstrate in this post using pure C++11.

如果要概括 weakly-ordered CPU，那就是一个 CPU 内核可以看到共享内存中值的变化顺序与另一个内核写入它们的顺序不同。 这就是我想在这篇文章中使用纯 C++11 演示的内容。

For normal applications, the x86/64 processor families from Intel and AMD do not have this characteristic. So we can forget about demonstrating this phenomenon on pretty much every modern desktop or notebook computer in the world. What we really need is a weakly-ordered multicore device. Fortunately, I happen to have one right here in my pocket:

对于一般的应用程序来说，Intel 和 AMD 的 x86/64 处理器家族都不具备这个特性。 因此我们别想着在现有的桌面系统或者笔记本上来阐述这种问题了。 我们需要的只是一个weakly-ordered的多核设备。 幸运的是，我的口袋里正好有一个：

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\iphone-4s.jpg)

The iPhone 4S fits the bill. It runs on a **dual-core ARM-based** processor, and the ARM architecture is, in fact, weakly-ordered.

Iphone4s就能满足要求。它运行在一个多核ARM处理器上，而这种ARM架构就是weakly-ordered.

## The Experiment

Our experiment will consist of an single integer, `sharedValue`, protected by a mutex. We’ll spawn two threads, and each thread will run until it has incremented `sharedValue` 10000000 times.

我们的实验由一个整型变量`sharedValue`以及保护它的锁(mutex)组成。我们来生成两个线程，并且每个线程都会对`sharedValue`做递增操作1千万次。

We won’t let our threads block waiting on the mutex. Instead, each thread will loop repeatedly doing busy work (ie. just wasting CPU time) and attempting to lock the mutex at random moments. If the lock succeeds, the thread will increment `sharedValue`, then unlock. If the lock fails, it will just go back to doing busy work. Here’s some pseudocode:

我们不要让线程一直等待锁(mutex)而阻塞，而是让每个线程都会忙碌地重复做一些事情(比如只是等待CPU时钟)，并试着在随机的时间里加锁。如果加锁成功，那线程就会将`shareValue`递增，否则，就会返回到原处继续做一些忙碌地工作。下面是其伪代码:

```
count = 0
while count < 10000000:
    doRandomAmountOfBusyWork()
    if tryLockMutex():
        // The lock succeeded
        sharedValue++
        unlockMutex()
        count++
    endif
endwhile
```

With each thread running on a separate CPU core, the timeline should look something like this. Each red section represents a successful lock and increment, while the dark blue ticks represent lock attempts which failed because the other thread was already holding the mutex.

每个线程都运行在不同的CPU核中，timeline应该是像下图这样的。每个红色区域代表一个成功的上锁和递增操作，深蓝色则代表加锁失败，因为这时另一个线程已经拥有这把锁了。

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\experiment-timeline.png)

It bears repeating that [a mutex is just a concept](http://preshing.com/20111124/always-use-a-lightweight-mutex), and there are [many](http://preshing.com/20120226/roll-your-own-lightweight-mutex) [ways](http://preshing.com/20120305/implementing-a-recursive-mutex) to implement one. We could use the implementation provided by `std::mutex`, and of course, everything will function correctly. But then I’d have nothing to show you. Instead, let’s implement a custom mutex – then let’s break it to demonstrate the consequences of [weak hardware ordering](http://preshing.com/20120930/weak-vs-strong-memory-models). Intuitively, the potential for memory reordering will be highest at those moments when there is a “close shave” between threads – for example, at the moment circled in the above diagram, when one thread acquires the lock *just* as the other thread releases it.

需要强调的是，[锁只是一个概念](http://www.chongh.wiki/blog/2016/06/27/lightweightmutex/)，有[许多](http://www.chongh.wiki/blog/2016/07/04/writelightweightmutex/)实现锁的方法。我们采用std::mutex中提供的实现方法，当然了，一切都会正常运行。倘若真是那样，我就没什么可以跟你们讲的了。不如这样，我们来实现一个常规的锁-我们来看看[弱硬件执行顺序](http://www.chongh.wiki/blog/2016/10/30/memorymodel/)的结果。直观上来看，当在线程间有个“close shave”时，内存乱序发生的可能性是最高的。举个例子，在上图中，当一个线程正好就在另一个线程释放锁的那一时刻获取锁，这种现象就叫做”close shave”。

The latest version of Xcode has terrific support for C++11 threads and atomic types, so let’s use those. All C++11 identifiers are defined in the `std` namespace, so let’s assume `using namespace std;` was placed somewhere earlier in the code.

Xcode的最新版本对C++11线程和原子类型支持非常好，那我们就用它吧。所有的C++标识符都定义在`std`命名空间中，我们假设`using namespace std`提前定义在了代码中某处。

## A Ridiculously Simple Mutex（一把诡异的锁）

Our mutex will consist of a single integer `flag`, where 1 indicates that the mutex is held, and 0 means it isn’t. To ensure mutual exclusivity, a thread can only set `flag` to 1 if the previous value was 0, and it must do so atomically. To achieve this, we’ll define `flag` as a C++11 atomic type, `atomic<int>`, and use a [read-modify-write](http://preshing.com/20120612/an-introduction-to-lock-free-programming#atomic-rmw) operation:

我们的锁由一个整型`flag`组成，1表示锁被持有，0表示不被持有。为了保证锁的独占权，一个线程只能在`flag`值为0的时候才能将其设置成1，而且这种操作必须保证原子性。为了做到这些，我们将`flag`定义为C++11的原子类型，`atomic<int>`，并采用[read-modify-write](http://preshing.com/20120612/an-introduction-to-lock-free-programming/#atomic-rmw)操作:

```
int expected = 0;
if (flag.compare_exchange_strong(expected, 1, memory_order_acquire))
{
    // The lock succeeded
}
```

The `memory_order_acquire` argument used above is considered an *ordering constraint*. We’re placing acquire semantics on the operation, to help guarantee that we receive the latest shared values from the previous thread which held the lock.

上面代码中用的`memory_order_acquire`参数被认为是执行顺序约束的(ordering constraint）.我们在操作中采用acquire语义，用来保证我们能从持有锁的前一个线程中获取最新的共享值

To release the lock, we perform the following:

为了释放锁，我们执行下面的操作:

```
flag.store(0, memory_order_release);
```

This sets `flag` back to 0 using the `memory_order_release` ordering constraint, which applies release semantics. [Acquire and release semantics](http://preshing.com/20120913/acquire-and-release-semantics) must be used as a pair to ensure that shared values propagate completely from one thread to the next.

上面操作使用`memory_order_release`执行顺序约束将`flag`设置回0了，其采用了release语义。[Acquire与Release语义](http://www.chongh.wiki/blog/2016/09/28/acquireandrelease/)必须成对使用,用来确保共享值能从一个线程中传播到另一个线程中注3。

## If We Don’t Use Acquire and Release Semantics…

Now, let’s write the experiment in C++11, but instead of specifying the correct ordering constraints, let’s put `memory_order_relaxed` in both places. This means no particular memory ordering will be enforced by the C++11 compiler, and [any kind of reordering](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) is permitted.

现在，我们不限定任何的正确执行顺序约束，使用C++11重写上述实验,只要在两处都放置’memory_order_relaxed`。这意味着C++ 11编译器不会强制任何特殊的内存执行顺序，[任何类型的乱序](http://www.chongh.wiki/blog/2016/09/19/sourcecontrol/)都是允许的。

```
void IncrementSharedValue10000000Times(RandomDelay& randomDelay)
{
    int count = 0;
    while (count < 10000000)
    {
        randomDelay.doBusyWork();
        int expected = 0;
        if (flag.compare_exchange_strong(expected, 1, memory_order_relaxed))
        {
            // Lock was successful
            sharedValue++;
            flag.store(0, memory_order_relaxed);
            count++;
        }
    }
}
```

At this point, it’s informative to look at the resulting ARM assembly code generated by the compiler, in Release, using the Disassembly view in Xcode:

这时，看看编译器产生的ARM汇编代码能告诉我们很多信息，在Release中，使用Xcode中的反汇编视图:

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\disasm-no-barriers.png)

If you aren’t very familiar with assembly language, don’t worry. All we want to know is whether the compiler has reordered any operations on shared variables. This would include the two operations on `flag`, and the increment of `sharedValue` in between. Above, I’ve annotated the corresponding sections of assembly code. As you can see, we got lucky: The compiler chose *not* to reorder those operations, even though the `memory_order_relaxed` argument means that, in all fairness, it could have.

如果你对汇编不是很熟悉，也不必担心。所有需要知道的就是编译器是否在共享变量中对任何操作重新排序了。这里包括对`flag`的两种操作以及对`sharedValue`的递增操作。在上图中，我标明了相应区域对应的汇编代码。正如你所知，我们是幸运的：尽管`memory_order_relaxed`参数意味着编译器可以对那些操作重新排序，但编译器并没有这么做.

I’ve put together a sample application which repeats this experiment indefinitely, printing the final value of `sharedValue` at the end of each trial run. It’s [available on GitHub](https://github.com/preshing/AcquireRelease) if you’d like to view the source code or run it yourself.

我已经给出了一个样例程序来无限循环这个实验，并在每一次运行结束后将`sharedValue`的最终值打印出来。如果你想看源码并自己运行，可以在[github](https://github.com/preshing/AcquireRelease)上下载。

Here’s the iPhone, hard at work, running the experiment:

下面是iphone正在努力的运行这个实验

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\iphone-running.jpg)

And here’s the output from the Output panel in Xcode:

这是Xcode输出框中的输出:

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\output-no-barriers.png)

Check it out! The final value of `sharedValue` is consistently less than 20000000, even though both threads perform exactly 10000000 increments, and the order of assembly instructions exactly matches the order of operations on shared variables as specified in C++.

动手检验一下！尽管两个线程都分别执行了递增操作1千万次,`sharedValue`的最终值一直小于2千万。并且，汇编指令的执行顺序和C++中定义在共享变量中的执行顺序操作是一致的。

As you might have guessed, these results are entirely due to memory reordering **on the CPU**. To point out just one possible reordering – and there are several – the memory interaction of `str.w r0, [r11]` (the store to `sharedValue`) could be reordered with that of `str r5, [r6]` (the store of 0 to `flag`). In other words, the mutex could be effectively unlocked before we’re finished with it! As a result, the other thread would be free to wipe out the change made by this one, resulting in a mismatched `sharedValue` count at the end of the experiment, just as we’re seeing here.

你可能会觉得，这些结果可能是因为CPU上的内存乱序导致的。为了指出一种可能的乱序-实际上有很多种可能-`str.w r0, [r11]`(sharedValue写操作)与内存的交互可能会和`str r5, [r6]`(将flag写入0）发生乱序。换句话说，在程序结束之前，锁能被有效地释放。正如我们所见，结果是另一个线程可以自由的擦除由这个线程对值的修改，导致在实验结束时`sharedValued`的不一致。

## Using Acquire and Release Semantics Correctly

Fixing our sample application, of course, means putting the correct C++11 memory ordering constraints back in place:

要修正上述的例子，意味着要重新放回正确的C++11内存执行顺序约束

```
void IncrementSharedValue10000000Times(RandomDelay& randomDelay)
{
    int count = 0;
    while (count < 10000000)
    {
        randomDelay.doBusyWork();
        int expected = 0;
        if (flag.compare_exchange_strong(expected, 1, memory_order_acquire))
        {
            // Lock was successful
            sharedValue++;
            flag.store(0, memory_order_release);
            count++;
        }
    }
}
```

As a result, the compiler now inserts a couple of `dmb ish` instructions, which act as memory barriers in the ARMv7 instruction set. I’m not an ARM expert – comments are welcome – but it’s safe to assume this instruction, much like `lwsync` on PowerPC, provides all the [memory barrier types](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) needed for acquire semantics on `compare_exchange_strong`, and release semantics on `store`.

结果，编译器现在插入一系列`dmb ish`指令，这些指令在ARMv7指令集中充当内存屏障。我不是个ARM专家–欢迎评论–但可以保险的假设这个指令很像PowerPC上的`lwsync`指令，为`compare_exchange_strong`上的acquire语义和`store`上的release语义提供了所有[内存屏障类型](http://www.chongh.wiki/blog/2016/09/19/sourcecontrol/)

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\disasm-with-barriers.png)

This time, our little home-grown mutex really does protect `sharedValue`, ensuring all modifications are passed safely from one thread to the next each time the mutex is locked.

这次，我们可爱的自制锁的确保护了`sharedValue`，确保每次加锁时所有的修改都从一个线程中安全的传播到了另一个线程中。

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\output-with-barriers.png)

If you still don’t grasp intuitively what’s going on in this experiment, I’d suggest a review of my [source control analogy](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) post. In terms of that analogy, you can imagine two workstations each having local copies of `sharedValue` and `flag`, with some effort required to keep them in sync. Personally, I find visualizing it this way very helpful.

如果你仍然无法理解这个实验发生了什么。我建议你看看[这篇](http://www.chongh.wiki/blog/2016/09/19/sourcecontrol/)文章。就类比而言，你可以想象一下，两个工作站其中每个都有`sharedValue`和`flag`的本地副本，需要一些其它的工作来确保它们是同步的。个人而言，用这种方法将其可视化是很有帮助的

I’d just like to reiterate that the memory reordering we saw here can only be observed on a **multicore** or multiprocessor device. If you take the same compiled application and run it on an iPhone 3GS or first-generation iPad, which use the same ARMv7 architecture but have only a single CPU core, you won’t see any mismatch in the final count of `sharedValue`.

我想重述一遍，我们在这里看到的内存乱序只能在多核或多处理器设备中才能发生。如果你把这个编译好的程序运行在Iphone3GS或更早一代的ipad中，它们都使用同样的ARMv7架构但只有一个CPU核，你也不会看到`sharedValue`最后结果的任何不匹配性。

## Interesting Notes

You can build and run [this sample application](https://github.com/preshing/AcquireRelease) on any Windows, MacOS or Linux machine with a x86/64 CPU, but unless your compiler performs reordering on specific instructions, you won’t witness any memory reordering at runtime – even on a multicore system! Indeed, when I tested it using Visual Studio 2012, no memory reordering occurred. That’s because x86/64 processors are what is usually considered [strongly-ordered](http://preshing.com/20120930/weak-vs-strong-memory-models#strong): When one CPU core performs a sequence of writes, every other CPU core sees those values change in the same order that they were written.

你可以把这个[示例程序](https://github.com/preshing/AcquireRelease)运行在任何x86/x64架构上的Window,MacOS或Linux机器上。就算是在一个多核系统中，除非是你的编译器在一些特殊指令上执行了乱序，否则你是不会看到任何运行时的内存乱序的。当我用Visual Studio2012测试时，并没有发生内存乱序。那是因为x86/x64处理器通常都被认为是[strongly-ordered](http://www.chongh.wiki/blog/2016/10/30/memorymodel/)：当一个CPU核执行一系列写操作时，每个其它CPU核都会看到它们在被写入时顺序的值变化

![img](F:\Git\programming_language\cpp\笔记\memory_model\this_is_why_they_call_it_a_Weakly-Ordered_CPU.assets\no-barrier-on-intel.png)

This goes to show how easy it is to use C++11 atomics incorrectly without knowing it, simply because it appears to work correctly on a specific processor and toolchain.

上图为了说明如果不了解C++11原子类型，就很容易错误的使用。很简单，因为它只有在一个特殊的处理器和工具链中才能正确运行

Incidentally, the release candidate of Visual Studio 2012 generates rather poor x86 machine code for this sample. It’s nowhere near as efficient as the ARM code generated by Xcode. Meanwhile, performance is the main reason to use lock-free programming on multicore in the first place! It’s enough to turn me off using C++11 atomics on Windows for the time being. *[**Update Feb. 2013**: As mentioned in the [comments](http://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu#comment-61574), the latest version of VS2012 Professional now generates much better machine code.]*

碰巧的是，Visual Studio2012的发行版对这个示例程序会产生非常糟糕的x86机器代码。在多核系统上使用lock-free编程，提高性能是最首要原因。现在有足够的理由让我无法忍受在Window平台上用C++11原子类型了。正如[评论](http://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu/#comment-61574)中提到的，VS2012专业版的最新版本产生的机器代码会好很多了

This post is a followup to an earlier post where I demonstrated [StoreLoad reordering](http://preshing.com/20120515/memory-reordering-caught-in-the-act) on x86/64. In my experience, however, the need for `#StoreLoad` does not come up quite as often in practice as the ordering constraints demonstrated here.

这篇博文是博文的后续。 然而，根据我的经验，#StoreLoad 的需求在实践中并不像这里展示的排序约束那样频繁。

这篇文章是之前我在 x86/64 上演示 [StoreLoad乱序](http://www.chongh.wiki/blog/2016/08/11/memoryreorder/) 重新排序的的续集。从我的经验来看，#StoreLoad在实际运用中并不如这里提到的执行顺序约束用的频繁

Finally, I’m not the first person to demonstrate weak hardware ordering in practice, though I might be the first to demonstrate it using C++11. There are earlier posts by [Pierre Lebeaupin](http://wanderingcoder.net/2011/04/01/arm-memory-ordering/) and [ridiculousfish](http://ridiculousfish.com/blog/posts/barrier.html) which use different experiments to demonstrate the same phenomenon.

最后，我不是第一个在实际运用当中阐述弱硬件执行顺序的人，但我可能是第一个使用C++11来阐述的人。之前[Pierre Lebeaupin](http://wanderingcoder.net/2011/04/01/arm-memory-ordering/)和[ridiculousfish](http://ridiculousfish.com/blog/posts/barrier.html)的一些文章采用了不同的实验来解释相同的现象。