# Acquire and Release Semantics

Generally speaking, in [lock-free programming](http://preshing.com/20120612/an-introduction-to-lock-free-programming), there are two ways in which threads can manipulate shared memory: They can compete with each other for a resource, or they can pass information co-operatively from one thread to another. Acquire and release semantics are crucial for the latter: reliable passing of information between threads. In fact, I would venture to guess that incorrect or missing acquire and release semantics is the #1 type of lock-free programming error.

一般来说，在无锁编程中，线程可以通过两种方式操作共享内存：它们可以相互竞争资源，或者它们可以协作地将信息从一个线程传递到另一个线程。 获取 (acquire) 和释放 (release) 语义对于后者至关重要：在线程之间可靠地传递信息。 事实上，我敢猜测不正确或缺失的获取 (acquire) 和释放 (release) 语义是#1 类型的无锁编程错误。

In this post, I’ll demonstrate various ways to achieve acquire and release semantics in C++. I’ll touch upon the C++11 atomic library standard in an introductory way, so you don’t already need to know it. And to be clear from the start, the information here pertains to lock-free programming *without* [sequential consistency](http://preshing.com/20120612/an-introduction-to-lock-free-programming#sequential-consistency). We’re dealing directly with memory ordering in a multicore or multiprocessor environment.

在这篇文章中，我会去探讨许多在C++中实现 Acquire 与 Release 语义的方法。还会简单介绍一下C++11原子库标准。所以，你事先不必具备这方面的知识。简明起见，这里的讨论仅限于非[顺序一致性](http://preshing.com/20120612/an-introduction-to-lock-free-programming/#sequential-consistency)的无锁编程。我们要关注的是多核或者多处理器环境下的内存执行顺序。

Unfortunately, the terms *acquire and release semantics* appear to be in even worse shape than the term *lock-free*, in that the more you scour the web, the more seemingly contradictory definitions you’ll find. Bruce Dawson offers a couple of good definitions (credited to Herb Sutter) about halfway through [this white paper](http://msdn.microsoft.com/en-us/library/windows/desktop/ee418650.aspx). I’d like to offer a couple of definitions of my own, staying close to the principles behind C++11 atomics:

不幸的是，你会发现Acquire与Release语义甚至比lock-free更难理解。关于这个词，如果你在网上搜索的越多，越会感觉与它的定义更矛盾。 得益于Herb Sutter，Bruce Dawson通过[white paper](https://msdn.microsoft.com/en-us/library/windows/desktop/ee418650.aspx)提供了一些好的定义。紧密结合C++11原子性背后的原则，我想给出一些我自己的定义。

> ![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\read-acquire.png)
>
> **Acquire semantics** is a property that can only apply to operations that **read** from shared memory, whether they are [read-modify-write](http://preshing.com/20120612/an-introduction-to-lock-free-programming#atomic-rmw) operations or plain loads. The operation is then considered a **read-acquire**. Acquire semantics prevent memory reordering of the read-acquire with any read or write operation that **follows** it in program order.
>
> Acquire语义的性质只能应用于共享内存中的读操作，不管是read-modify-write操作还是普通的读数据。这种操作被认为是read-acquire。 Acquire 语义能阻止在程序序中出现在 read-acquire 和它之后的任何读写操作的乱序。

> ![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\write-release.png)
>
> **Release semantics** is a property that can only apply to operations that **write** to shared memory, whether they are read-modify-write operations or plain stores. The operation is then considered a **write-release**. Release semantics prevent memory reordering of the write-release with any read or write operation that **precedes** it in program order.
>
> Release语义的性质只能应用于共享内存中的写操作，而不管是read-modify-write操作还是普通的写操作。这种操作被认为是write-release. Release 语义能阻止程序序中出现在 write-release 和它之前的任何读写操作的乱序。

Once you digest the above definitions, it’s not hard to see that acquire and release semantics can be achieved using simple combinations of the memory barrier types I [described at length in my previous post](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations). The barriers must (somehow) be placed *after* the read-acquire operation, but *before* the write-release. *[Update: Please note that these barriers are technically more strict than what’s required for acquire and release semantics on a single memory operation, but they do achieve the desired effect.]*

只要你能消化上述的概念，就不难知道 Acquire 与 Release 语义可以通过我在[上一篇文章中](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations)提到的memory barrier类型的组合来实现。Barrier必须(以某种方式)放置在 read-acquire 操作之后与 write-release 操作之前。*[更新：请注意这些barrier在技术上比单个内存操作上对 Acquire 与 Release 语义的需求更加严格，但能达到理想中的效果]*

![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\acq-rel-barriers.png)

What’s cool is that neither acquire nor release semantics requires the use of a `#StoreLoad` barrier, which is often a more expensive memory barrier type. For example, on PowerPC, the `lwsync` (short for “lightweight sync”) instruction acts as all three `#LoadLoad`, `#LoadStore` and `#StoreStore` barriers at the same time, yet is less expensive than the `sync` instruction, which includes a `#StoreLoad` barrier.

有趣的是不管是 Acquire 还是 Release 语义都不需要用到 #StoreLoad 这种开销比较昂贵的memory barrier 。举个例子，在PowerPC中，lwsync( “lightweight sync” 的简写)指令同时充当 #LoadLoad , #LoadStore 和 #StoreStore 三种屏障，所以比 sync 指令(包含了 #StoreLoad 屏障)的开销要低.

## With Explicit Platform-Specific Fence Instructions

One way to obtain the desired memory barriers is by issuing explicit fence instructions. Let’s start with a simple example. Suppose we’re coding for PowerPC, and `__lwsync()` is a compiler intrinsic function that emits the `lwsync` instruction. Since `lwsync` provides so many barrier types, we can use it in the following code to establish either acquire or release semantics as needed. In Thread 1, the store to `Ready` turns into a write-release, and in Thread 2, the load from `Ready` becomes a read-acquire.

获取所需内存屏障的一种方式就是发出明确的栅栏指令。我们以一个简单的例子开始。假设我们在PowerPC平台下写代码，__lwsync()是一个编译器内置函数，能发出 lwsync 指令。由于 lwsync 提供了许多种内存屏障，因此我们可以在下面的代码中用它来建立所需的 Acquire 或者 Release 语义。在线程1中，对 Ready 的写操作变成了一个write-release，在线程2中，对 Ready 的读操作变成了一个 read-acquire。

![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\platform-fences.png)



[![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\analogy-small.png)](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations)



If we let both threads run and find that `r1 == 1`, that serves as confirmation that the value of `A` assigned in Thread 1 was passed successfully to Thread 2. As such, we are guaranteed that `r2 == 42`. In my previous post, I already [gave a lengthy analogy](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) for `#LoadLoad` and `#StoreStore` to illustrate how this works, so I won’t rehash that explanation here.

如果我们让两个线程都运行并发现 r1 == 1，则可以确认线程 1 中分配的 A 的值已成功传递给线程 2。因此，我们保证 r2 == 42。在我之前的帖子中 ，我已经对#LoadLoad 和 #StoreStore 做了一个冗长的类比来说明这是如何工作的，所以我不会在这里重复这个解释。

In formal terms, we say that the store to `Ready` *synchronized-with* the load. I’ve written a separate post about *synchronizes-with* [here](http://preshing.com/20130823/the-synchronizes-with-relation). For now, suffice to say that for this technique to work in general, the acquire and release semantics must apply to the same variable – in this case, `Ready` – and both the load and store must be atomic operations. Here, `Ready` is a simple aligned `int`, so the operations are already atomic on PowerPC.

在正式的定义中，我们说对 Ready 的存储操作同步于加载操作。在[这里](http://preshing.com/20130823/the-synchronizes-with-relation/)我对`synchronizes-with`专门写了一篇文章。目前为止，我们可以说要让这技术能通用，Acquire 与 Release 语义必须应用在同一个变量中（在这个例子中是Ready变量），读和写操作必须都是原子操作。在这里，Ready是个简单的已经对齐的整型变量，所以这些读写在PowerPC中就已经是原子操作了。

## With Fences in Portable C++11

The above example is compiler- and processor-specific. One approach for supporting multiple platforms is to convert the code to C++11. All C++11 identifiers exist in the `std` namespace, so to keep the following examples brief, let’s assume the statement `using namespace std;` was placed somewhere earlier in the code.

上述的例子是依赖编译器和处理器的。要支持多平台的一种方法就是将代码转化成C++11. 所有的C++11标识符都存在std空间中，所以为了保持下面例子的简洁，我们假设`using namespace std;`语句提前放在代码的某处了。

C++11’s atomic library standard defines a portable function `atomic_thread_fence()` that takes a single argument to specify the type of fence. There are several possible values for this argument, but the values we’re most interested in here are `memory_order_acquire` and `memory_order_release`. We’ll use this function in place of `__lwsync()`.

C++11的原子库标准定义了一个可移植的函数`atomic_thread_fence()`，函数采用一个参数来指定fence的类型。这个参数有很多种可能的值，但我们在这里最感兴趣的是`memory_order_acquire`与`memory_order_release`. 我们用这个函数来替代__lwsync()。

There’s one more change to make before this example is complete. On PowerPC, we knew that both operations on `Ready` were atomic, but we can’t make that assumption about every platform. To ensure atomicity on all platforms, we’ll change the type of `Ready` from `int` to `atomic<int>`. I know, it’s kind of a silly change, considering that aligned loads and stores of `int` are already atomic on every modern CPU that exists today. I’ll write more about this in the post on [*synchronizes-with*](http://preshing.com/20130823/the-synchronizes-with-relation), but for now, let’s do it for the warm fuzzy feeling of 100% correctness in theory. No changes to `A` are necessary.

在让这个例子变得更完整之前，还需要作一点修改。在PowerPC平台上，我们知道对Ready变量的两个操作都是原子的，但不是每个平台上都这样。我们可以将Ready变量从整型改为`atomic<int>`。 考虑到针对对齐的整型，读写操作在现今所有的CPU中都是原子的，我知道这是个傻瓜式的修改。我会在synchronizes-with文章中描述更多关于这方面的内容，现在，我们姑且认为理论上能确保100%的准确率。另外， 不必对A做任何修改。

![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\cpp11-fences.png)

The `memory_order_relaxed` arguments above mean “ensure these operations are atomic, but don’t impose any ordering constraints/memory barriers that aren’t already there.”

`memory_order_relaxed` 参数意味着“确保这些操作是原子的，但对那些本身不在那里的操作不作任何的顺序限制或者强加内存屏障”

Once again, both of the above `atomic_thread_fence()` calls can be (and hopefully are) implemented as `lwsync` on PowerPC. Similarly, they could both emit a `dmb` instruction on ARM, which I believe is at least as effective as PowerPC’s `lwsync`. On x86/64, both `atomic_thread_fence()` calls can simply be implemented as [compiler barriers](http://preshing.com/20120625/memory-ordering-at-compile-time), since *usually*, every load on x86/64 already implies acquire semantics and every store implies release semantics. This is why x86/64 is often said to be [strongly ordered](http://preshing.com/20120930/weak-vs-strong-memory-models).

再说一次，上述的两个 `atomic_thread_fence()` 调用都可以（并且希望）实现像在PowerPC中 lwsync 一样的效果。类似的，他们都能在ARM上发出 dmb 指令，这点我相信至少是和在PowerPC平台上能有同样效果的。在X86/X64平台上，`atomic_thread_fence()`调用可以简单的实现成和 compiler barrier一样的效果，因为一般来说，x86/x64上的每个读操作已经包含了Acquire语义，并且每个写操作都包含了Release 语义。这就是为什么x86/x64经常被说成是强内存模型

## Without Fences in Portable C++11

In C++11, it’s possible to achieve acquire and release semantics on `Ready` without issuing explicit fence instructions. You just need to specify memory ordering constraints directly on the operations on `Ready`:

在C++11中，只要在对Ready上的操作中指定内存执行顺序的限制，就可以不发出显式的栅栏指令来实现 Acquire 和 Release 语义。

![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\cpp11-no-fences.png)

Think of it as rolling each fence instruction into the operations on `Ready` themselves. *[Update: Please note that this form is [not exactly the same](http://preshing.com/20131125/acquire-and-release-fences-dont-work-the-way-youd-expect) as the version using standalone fences; technically, it’s less strict.]* The compiler will emit any instructions necessary to obtain the required barrier effects. In particular, on Itanium, each operation can be easily implemented as a single instruction: `ld.acq` and `st.rel`. Just as before, `r1 == 1` indicates a *synchronizes-with* relationship, serving as confirmation that `r2 == 42`.

考虑每个在Ready上的栅栏指令。[更新:请注意这种形式和使用独立的fences版本是不完全一样的，技术上来说，没有那么严格]。 编译器会发出必要的指令，来取得和屏障一样的所需的效果。具体来说，在Itanium上，每个操作都能简单的实现成一个单指令:`ld.acq` 与 `st.rel`. 如之前那样，r1 == 1意味着一种
synchronizes-with关系，作为对r2 == 42的确认。

This is actually the preferred way to express acquire and release semantics in C++11. In fact, the `atomic_thread_fence()` function used in the previous example was [added relatively late](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2633.html) in the creation of the standard.

在C++11中，这实际上是一种表达Acquire与 Release语义的首选方式。前面例子中使用的`atomic_thread_fence()` 函数在标准的制定中是添加的相对较晚的。

## Acquire and Release While Locking

As you can see, none of the examples in this post took advantage of the `#LoadStore` barriers provided by acquire and release semantics. Really, only the `#LoadLoad` and `#StoreStore` parts were necessary. That’s just because in this post, I chose a simple example to let us focus on API and syntax.

如你所见，这篇文章中没有一个例子能利用 Acquire与Release 语义提供的 #LoadStore 屏障的优势。 实际上，只有 #LoadLoad 和 #StoreStore 就足矣。这就是为什么在这篇文章中，我选一个简单的列子来，这样能让我们集中关注API和语义。

One case in which the `#LoadStore` part becomes essential is when using acquire and release semantics to implement a (mutex) lock. In fact, this is where the names come from: acquiring a lock implies acquire semantics, while releasing a lock implies release semantics! All the memory operations in between are contained inside a nice little barrier sandwich, preventing any undesireable memory reordering across the boundaries.

\#LoadStore 部分变得必不可少的一种情况是使用 acquire 和 release 语义来实现（互斥）锁。 事实上，这就是名称的由来：获取锁意味着获取语义，而释放锁意味着释放语义！ 中间的所有内存操作都包含在一个漂亮的小屏障三明治中，防止任何不受欢迎的内存跨边界重新排序。

![img](F:\Git\programming_language\cpp\笔记\memory_model\acquire_and_release_sem.assets\acq-rel-lock.png)

Here, acquire and release semantics ensure that all modifications made while holding the lock will propagate fully to the next thread that obtains the lock. Every implementation of a lock, even one you [roll on your own](http://preshing.com/20120226/roll-your-own-lightweight-mutex), should provide these guarantees. Again, it’s all about passing information reliably between threads, especially in a multicore or multiprocessor environment.

在这里，acquire 和 release 语义确保在持有锁时所做的所有修改将完全传播到下一个获得锁的线程。 锁的每一个实现，即使是你自己滚动的，都应该提供这些保证。 同样，这一切都是关于在线程之间可靠地传递信息，尤其是在多核或多处理器环境中。

In a followup post, I’ll show a [working demonstration](http://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu) of C++11 code, running on real hardware, which can be plainly observed to break if acquire and release semantics are not used.

在后续文章中，我将展示在真实硬件上运行的 C++11 代码的工作演示，如果不使用获取和释放语义，可以清楚地观察到它会中断。

