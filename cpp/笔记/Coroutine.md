# Coroutine Theory （协程理论）

- 原文地址：https://lewissbaker.github.io/2017/09/25/coroutine-theory

This is the first of a series of posts on the [C++ Coroutines TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf), a new language feature that is currently on track for inclusion into the C++20 language standard.

这是C++ Coroutine TS 系列文章的第一篇，C++ Coroutine TS 是一种新的语言特性，目前有望纳入C++ 20 语言标准。

In this series I will cover how the underlying mechanics of C++ Coroutines work as well as show how they can be used to build useful higher-level abstractions such as those provided by the [cppcoro](https://github.com/lewissbaker/cppcoro) library.

在本系列文章里，我将介绍C++ 协程的基本机制如何工作，并展示如何利用其构建类似 cppcoro 这样实用的高抽象级别的库。

In this post I will describe the differences between functions and coroutines and provide a bit of theory about the operations they support. The aim of this post is introduce some foundational concepts that will help frame the way you think about C++ Coroutines.

在这篇文章中，我将描述函数和协程之间的区别，并介绍一些他们所支持的操作的理论。这篇文章的目的是介绍一些概念，这些概念将有助于构建你关于C++协程的思考方式。

## Coroutine  are Functions are Coroutine （协程是函数，函数也是协程）

A coroutine is a generalisation of a function that allows the function to be suspended and then later resumed.

协程是对函数的泛化，它允许函数被挂起，并在之后恢复执行。

I will explain what this means in a bit more detail, but before I do I want to first review how a “normal” C++ function works.

我将详细介绍（ a bit of theory ）意味着什么，但是在此之前呢，我想先回顾一下“普通” C++ 函数是如何工作的。

## “Normal” Functions （“普通”函数）

A normal function can be thought of as having two operations: **Call** and **Return** (Note that I’m lumping “throwing an exception” here broadly under the **Return** operation).

普通函数可以被认为有两个操作：调用和返回( **Call** and **Return** )（请注意，这里我将“抛出异常”概括地归结为返回操作下面）。

The **Call** operation creates an activation frame, suspends execution of the calling function and transfers execution to the start of the function being called.

**Call** 操作将会创建一个活动帧，挂起( **suspends**)调用函数的执行，并将执行流转移到被调用函数的开头。

The **Return** operation passes the return-value to the caller, destroys the activation frame and then resumes execution of the caller just after the point at which it called the function.

**Return** 操作将返回值(**return-value**)传递给调用者，销毁活动帧并将执行流交还给调用者，调用者从调用函数的位置后恢复执行。

Let’s analyse these semantics a little more…

让我们再分析一下这些语义。。。

### Activation Frames（活动帧）

So what is this ‘activation frame’ thing?

那么什么是“活动帧” 呢？

You can think of the activation frame as the block of memory that holds the current state of a particular invocation of a function. This state includes the values of any parameters that were passed to it and the values of any local variables.

你可以将活动帧视为一个内存块，它维护了调用函数的当前状态。这些状态包括传递给它的所有的参数值和局部变量。

For “normal” functions, the activation frame also includes the return-address - the address of the instruction to transfer execution to upon returning from the function - and the address of the activation frame for the invocation of the calling function. You can think of these pieces of information together as describing the ‘continuation’ of the function-call. ie. they describe which invocation of which function should continue executing at which point when this function completes.

对于“普通”函数而言，活动帧还包含返回地址(**return-address**): \- 从函数返回时将执行流转移到的指令的地址 - 以及用于调用函数的活跃帧的地址。你可以认为这些信息一起描述了函数调用的 “continuation（延续）”，即它们描述了当一个函数完成时，应该在哪个位置继续执行哪一个函数的调用。

With “normal” functions, all activation frames have strictly nested lifetimes. This strict nesting allows use of a highly efficient memory allocation data-structure for allocating and freeing the activation frames for each of the function calls. This data-structure is commonly referred to as “the stack”.

对于“普通”函数来说，所有的活动帧都拥有严格的嵌套生命周期。这种严格的嵌套，允许使用高效的内存分配数据结构来为每个函数调用分配和释放活动帧。这种数据结构通常称为“栈”。

When an activation frame is allocated on this stack data structure it is often called a “stack frame”.

当一个活动帧被分配在栈上时，它通常被称为“栈帧”。

This stack data-structure is so common that most (all?) CPU architectures have a dedicated register for holding a pointer to the top of the stack (eg. in X64 it is the `rsp` register).

这种栈结构非常普遍，以至于大多数（或者全部？）CPU 体系架构都有一个专门的寄存器，用于保存指向栈顶部的指针（例如，在X64中，它是rsp寄存器）。

To allocate space for a new activation frame, you just increment this register by the frame-size. To free space for an activation frame, you just decrement this register by the frame-size.

当为一个新的活动帧申请空间时，你只需要将该寄存器按帧大小递增即可。要释放活动帧的空间时，只要将该寄存器按帧大小减小即可。

### The ‘Call’ Operation

When a function calls another function, the caller must first prepare itself for suspension.

当一个函数调用另外一个函数时，调用者(**Caller**) 必须首先为暂停自身执行做准备。

This ‘suspend’ step typically involves saving to memory any values that are currently held in CPU registers so that those values can later be restored if required when the function resumes execution. Depending on the calling convention of the function, the caller and callee may coordinate on who saves these register values, but you can still think of them as being performed as part of the **Call** operation.

“暂停”步骤通常是将当前保存在 CPU 寄存器的值保存到内存中，以便在函数恢复执行时可以根据需要还原这些值。取决于函数的调用约定，调用者和被调用者可以协商谁保存这些寄存器值，但是你仍可以将它们视为 **Call** 操作的一部分。

The caller also stores the values of any parameters passed to the called function into the new activation frame where they can be accessed by the function.

调用者还会将传递给被调用函数的任何参数值存储到新的活动帧中，以便在被调用函数中访问。

Finally, the caller writes the address of the resumption-point of the caller to the new activation frame and transfers execution to the start of the called function.

最后，调用者将自己的恢复点地址写入新的活动帧，并将执行流转移到被调用函数的开头。

In the X86/X64 architecture this final operation has its own instruction, the `call` instruction, that writes the address of the next instruction onto the stack, increments the stack register by the size of the address and then jumps to the address specified in the instruction’s operand.

在 X86/X64 架构中，这个最后的操作有自己的指令，即 `Call` 指令，它将下一条指令地址写入栈，按地址大小增加栈寄存器，然后跳转到指令操作数中指定的地址。

### The ‘Return’ Operation

When a function returns via a `return`-statement, the function first stores the return value (if any) where the caller can access it. This could either be in the caller’s activation frame or the function’s activation frame (the distinction can get a bit blurry for parameters and return values that cross the boundary between two activation frames).

当函数通过 `return` 语句返回时，该函数会先将返回值（如果有）存储在调用者可以访问的地方。返回值可能存储在调用者或者函数的活动帧中（这样来区分，会让跨越两个活动帧的参数和返回值变得有点模糊）。

Then the function destroys the activation frame by:

- Destroying any local variables in-scope at the return-point.
- Destroying any parameter objects
- Freeing memory used by the activation-frame

然后，函数通过以下步骤销毁活动帧：

- 销毁返回点作用域内的所有局部变量。
- 销毁所有的参数对象
- 释放活动帧使用的内存

And finally, it resumes execution of the caller by:

- Restoring the activation frame of the caller by setting the stack register to point to the activation frame of the caller and restoring any registers that might have been clobbered by the function.
- Jumping to the resume-point of the caller that was stored during the ‘Call’ operation.

最后，它通过以下方式恢复调用者的执行：

- 通过将栈寄存器设置为指向调用者的活动帧来恢复调用者的活动帧，并恢复任何可能被调用函数破坏的寄存器。
- 跳转到 “**Call**” 操作期间，保存起来的调用者恢复点。

Note that as with the ‘Call’ operation, some calling conventions may split the repsonsibilities of the ‘Return’ operation across both the caller and callee function’s instructions.

注意，和 **Call** 操作一样，某些约定可能会在调用者和被调用者之间划分 **Return** 操作的责任。

## Coroutines

Coroutines generalise the operations of a function by separating out some of the steps performed in the **Call** and **Return** operations into three extra operations: **Suspend**, **Resume** and **Destroy**.

协程泛化了函数的操作，将 **Call** 和 **Return** 操作中执行的一些步骤又分成了三个额外的操作：**Suspend**，**Resume** 和 **Destroy**。

The **Suspend** operation suspends execution of the coroutine at the current point within the function and transfers execution back to the caller or resumer without destroying the activation frame. Any objects in-scope at the point of suspension remain alive after the coroutine execution is suspended.

**Suspend** 操作可以在函数内部将协程挂起，并在不破坏活动帧的情况下将执行流交还给调用者或者恢复者。协程挂起后，挂起点内的所有对象仍然是可用的。

Note that, like the **Return** operation of a function, a coroutine can only be suspended from within the coroutine itself at well-defined suspend-points.

注意，就像函数的 **Return** 操作，协程只能在定义好的暂停点、从协程内部暂停。

The **Resume** operation resumes execution of a suspended coroutine at the point at which it was suspended. This reactivates the coroutine’s activation frame.

**Resume** 操作恢复执行一个在挂起点挂起的协程。这将重新激活协程的活动帧。

The **Destroy** operation destroys the activation frame without resuming execution of the coroutine. Any objects that were in-scope at the suspend point will be destroyed. Memory used to store the activation frame is freed.

**Destroy** 操作销毁协程的活动帧并且不恢复协程的执行。挂起点范围内的所有对象都将被销毁。用于保存活动帧的内存也将被释放。

### Coroutine activation frames

Since coroutines can be suspended without destroying the activation frame, we can no longer guarantee that activation frame lifetimes will be strictly nested. This means that activation frames cannot in general be allocated using a stack data-structure and so may need to be stored on the heap instead.

由于协程可以在不破坏活动帧的情况下被挂起，我们不能再保证活动帧的生命周期被严格嵌套。这意味着活动帧通常不能像以前一样保存在栈中，因此可能需要将其存储在堆中。

There are some provisions in the C++ Coroutines TS to allow the memory for the coroutine frame to be allocated from the activation frame of the caller if the compiler can prove that the lifetime of the coroutine is indeed strictly nested within the lifetime of the caller. This can avoid heap allocations in many cases provided you have a sufficiently smart compiler.

如果编译器可以证明协程的生命周期确实严格嵌套在调用者的生命周期内，则 C++ Coroutine TS 中有一些规定允许从调用者的活动帧中为协程帧（coroutine frame）分配内存。这样，许多情况下就可以避免从堆中分配内存了。

With coroutines there are some parts of the activation frame that need to be preserved across coroutine suspension and there are some parts that only need to be kept around while the coroutine is executing. For example, the lifetime of a variable with a scope that does not span any coroutine suspend-points can potentially be stored on the stack.

对于协程来说，活动帧的某些部分是需要在被挂起时保存的，而有些部分只需要在协程执行时保持就行。例如，作用域内生命周期不跨越任何协程挂起点的变量可以保存在栈中。

You can logically think of the activation frame of a coroutine as being comprised of two parts: the ‘coroutine frame’ and the ‘stack frame’.

你可以从逻辑上认为协程的活动帧包含“协程帧”和“栈帧”两部分。

The ‘coroutine frame’ holds part of the coroutine’s activation frame that persists while the coroutine is suspended and the ‘stack frame’ part only exists while the coroutine is executing and is freed when the coroutine suspends and transfers execution back to the caller/resumer.

“协程帧”维护了一部分协程的活动帧，协程帧在协程被挂起时仍然存在，而“栈帧”仅在协程执行时存在，在协程被挂起并将执行流转移到调用者/恢复者时被释放。

### The ‘Suspend’ operation

The **Suspend** operation of a coroutine allows the coroutine to suspend execution in the middle of the function and transfer execution back to the caller or resumer of the coroutine.

协程的 **Suspend** 操作允许协程在函数中间暂停执行，并将执行流交还给协程的调用者或恢复者。

There are certain points within the body of a coroutine that are designated as suspend-points. In the C++ Coroutines TS, these suspend-points are identified by usages of the `co_await` or `co_yield` keywords.

协程内的某些点被称为挂起点。在 C++ Coroutine TS 中，这些挂起点是通过是`co_await` 或 `co_yield `关键字标识的。

When a coroutine hits one of these suspend-points it first prepares the coroutine for resumption by:

- Ensuring any values held in registers are written to the coroutine frame
- Writing a value to the coroutine frame that indicates which suspend-point the coroutine is being suspended at. This allows a subsequent **Resume** operation to know where to resume execution of the coroutine or so a subsequent **Destroy** to know what values were in-scope and need to be destroyed.

当协程到达这些挂起点时，它通过以下步骤为以后恢复协程做准备：

- 将寄存器中保存的值写入协程帧。
- 向协程帧中写入一个值，该值指示协程的挂起点。这使后续的 **Resume** 操作知道该从哪里恢复执行协程，或者让后续的 **Destroy** 操作知道在范围内的哪些值需要被销毁。

Once the coroutine has been prepared for resumption, the coroutine is considered ‘suspended’.

一旦协程为恢复做好了准备，协程就被认为是“挂起”的了。

The coroutine then has the opportunity to execute some additional logic before execution is transferred back to the caller/resumer. This additional logic is given access to a handle to the coroutine-frame that can be used to later resume or destroy it.

在执行流转回交给调用者/恢复者之前，协程有机会执行一些其他逻辑。这些额外逻辑被用来访问协程帧的句柄，该句柄可以在以后用来恢复或销毁协程。

This ability to execute logic after the coroutine enters the ‘suspended’ state allows the coroutine to be scheduled for resumption without the need for synchronisation that would otherwise be required if the coroutine was scheduled for resumption prior to entering the ‘suspended’ state due to the potential for suspension and resumption of the coroutine to race. I’ll go into this in more detail in future posts.

这种在协程进入“挂起的”状态后执行逻辑的能力允许将协程调度到恢复状态，而不需要同步，如果协程在进入 “挂起的” 状态之前被调度到执行恢复操作，则需要同步，这是因为协程有可能在挂起和恢复操作中产生潜在的竞争。我将在以后的文章中更详细地讨论这个问题。

The coroutine can then choose to either immediately resume/continue execution of the coroutine or can choose to transfer execution back to the caller/resumer.

协程可以选择立即恢复/继续执行协程，也可以选择将执行流转移到调用者/恢复者。

If execution is transferred to the caller/resumer the stack-frame part of the coroutine’s activation frame is freed and popped off the stack.

如果执行流被转交到调用者/恢复者，则释放协程活动帧的栈帧部分，并将其从栈中弹出。

### The ‘Resume’ operation

The **Resume** operation can be performed on a coroutine that is currently in the ‘suspended’ state.

可以在当前处于“挂起”状态的协程上执行**Resume** 操作。

When a function wants to resume a coroutine it needs to effectively ‘call’ into the middle of a particular invocation of the function. The way the resumer identifies the particular invocation to resume is by calling the `void resume()` method on the coroutine-frame handle provided to the corresponding **Suspend** operation.

当一个函数想要恢复协程时，它需要有效地“调用”到特定函数调用中间。恢复者标识要恢复的特定调用的方式是通过在提供给相应的 **Suspend** 操作的协程帧句柄上调用 `void resume()`方法。

Just like a normal function call, this call to `resume()` will allocate a new stack-frame and store the return-address of the caller in the stack-frame before transferring execution to the function.

就像普通函数调用一样，调用 `resume()` 将会分配一个新的栈帧，并在将执行流交到该函数之前将调用者的返回地址储存到栈帧中。

However, instead of transferring execution to the start of the function it will transfer execution to the point in the function at which it was last suspended. It does this by loading the resume-point from the coroutine-frame and jumping to that point.

但是，它不是将执行流移交到函数的开始，而是将执行流转移到上次的挂起点。这是通过从协程帧加载恢复点并跳转到这一点实现的。

When the coroutine next suspends or runs to completion this call to `resume()` will return and resume execution of the calling function.

当协程接下来挂起或执行完毕时，调用 `resume()` 将返回并恢复对调用函数的执行。

### The ‘Destroy’ operation

The **Destroy** operation destroys the coroutine frame without resuming execution of the coroutine.

**Destroy** 操作销毁协程帧，并且不恢复协程的执行。

This operation can only be performed on a suspended coroutine.

这个操作只能在挂起的协程上执行。

The **Destroy** operation acts much like the **Resume** operation in that it re-activates the coroutine’s activation frame, including allocating a new stack-frame and storing the return-address of the caller of the **Destroy** operation.

**Destroy** 操作与 **Resume** 操作非常相似，它会重新激活协程的活动帧，包括分配新的栈帧和存储 **Destroy** 操作调用者的返回地址。 

However, instead of transferring execution to the coroutine body at the last suspend-point it instead transfers execution to an alternative code-path that calls the destructors of all local variables in-scope at the suspend-point before then freeing the memory used by the coroutine frame.

然而，它不是将执行流移交给最后一次的挂起点，而是将执行权转交到另一个代码路径，该代码路径在挂起点作用域内调用所有局部变量的析构函数，然后释放协程帧使用的内存。

Similar to the **Resume** operation, the **Destroy** operation identifies the particular activation-frame to destroy by calling the `void destroy()` method on the coroutine-frame handle provided during the corresponding **Suspend** operation.

与 **Resume** 操作相似，**Destroy** 操作通过在相应的 **Suspend** 操作过程中提供的协程帧句柄上调用 `void destroy()` 函数来标识要销毁特定活动帧。

### The ‘Call’ operation of a coroutine

The **Call** operation of a coroutine is much the same as the call operation of a normal function. In fact, from the perspective of the caller there is no difference.

协程的 **Call** 操作和普通函数的 **Call** 操作基本相同。事实上，站在调用者的角度来说，两者完全相同。

However, rather than execution only returning to the caller when the function has run to completion, with a coroutine the call operation will instead resume execution of the caller when the coroutine reaches its first suspend-point.

但是，和函数仅在自己执行完毕时才将执行权交还给调用者不同，协程在到达第一个挂起点时，**Call**  操作将恢复调用者的执行。

When performing the **Call** operation on a coroutine, the caller allocates a new stack-frame, writes the parameters to the stack-frame, writes the return-address to the stack-frame and transfers execution to the coroutine. This is exactly the same as calling a normal function.

当在一个协程上执行 **Call** 操作时，调用者将分配一个新的栈帧，并将参数和调用者的返回地址写入栈帧，然后将执行流交给协程。这与调用一个普通函数完全相同。

The first thing the coroutine does is then allocate a coroutine-frame on the heap and copy/move the parameters from the stack-frame into the coroutine-frame so that the lifetime of the parameters extends beyond the first suspend-point.

协程要做的第一件事就是在堆中分配一个协程帧，然后将参数从栈帧复制/移动到协程帧，这样参数的生命周期就可以超出第一次的挂起点了。

### The ‘Return’ operation of a coroutine

The **Return** operation of a coroutine is a little different from that of a normal function.

协程的 **Return** 操作和普通函数的 **Return** 操作略有不同。

When a coroutine executes a `return`-statement (`co_return` according to the TS) operation it stores the return-value somewhere (exactly where this is stored can be customised by the coroutine) and then destructs any in-scope local variables (but not parameters).

当协程执行 `return` 语句（TS规范中的 `co_return`）操作时，它会将返回值储存到某个地方（协程可以自定义这个值的存储位置），然后销毁作用域内的任何局部变量（不是参数）。

The coroutine then has the opportunity to execute some additional logic before transferring execution back to the caller/resumer.

在将执行流转移到调用者/恢复者之前，协程可以执行一些额外的逻辑。

This additional logic might perform some operation to publish the return value, or it might resume another coroutine that was waiting for the result. It’s completely customisable.

这些额外的逻辑可以执行一些操作去发布返回值，或者恢复另外一个等待结果的协程。这是可自定义的。

The coroutine then performs either a **Suspend** operation (keeping the coroutine-frame alive) or a **Destroy** operation (destroying the coroutine-frame).

然后协程可以执行 **Suspend** 操作（保持协程帧存活）或者执行 **Destroy** 操作（销毁协程帧）。

Execution is then transferred back to the caller/resumer as per the **Suspend**/**Destroy** operation semantics, popping the stack-frame component of the activation-frame off the stack.

之后，按照 **Suspend** /**Destroy** 操作语义将执行流转交回调用者/恢复者，再从栈中弹出活动帧的栈帧。

It is important to note that the return-value passed to the **Return** operation is not the same as the return-value returned from a **Call** operation as the return operation may be executed long after the caller resumed from the initial **Call** operation.

需要注意的是，传递给 **Return** 操作的返回值和从 **Call** 操作返回的返回值是不一样的，因为 **Return** 操作可能在调用者从初始 **Call** 操作恢复很久之后才执行。

## An illustration

To help put these concepts into pictures, I want to walk through a simple example of what happens when a coroutine is called, suspends and is later resumed.

为了将这些概念更加形象的表示出来，我想通过一个简单的示例说明协程被调用、暂停和恢复时发生了什么。

So let’s say we have a function (or coroutine), `f()` that calls a coroutine, `x(int a)`.

假设我们有这么一个函数（或者协程）`f()`，它调用协程`x(int a)`。

Before the call we have a situation that looks a bit like this:

调用之前的情形大致如下：

```
	栈                     寄存器               堆

                          +------+
+---------------+ <------ | rsp  |
|  f()          |         +------+
+---------------+
| ...           |
|               |
```

Then when `x(42)` is called, it first creates a stack frame for `x()`, as with normal functions.

然后，当调用`x(42)`时，它像普通函数一样，先创建 `x()` 的栈帧。

```
	栈                     寄存器               堆
+----------------+ <-+
|  x()           |   |
| a  = 42        |   |
| ret= f()+0x123 |   |    +------+
+----------------+   +--- | rsp  |
|  f()           |        +------+
+----------------+
| ...            |
|                |
```

Then, once the coroutine `x()` has allocated memory for the coroutine frame on the heap and copied/moved parameter values into the coroutine frame we’ll end up with something that looks like the next diagram. Note that the compiler will typically hold the address of the coroutine frame in a separate register to the stack pointer (eg. MSVC stores this in the `rbp` register).

然后，一旦协程 `x()`在堆上为协程分配了内存，并将参数值复制/移动到协程帧后，我们将得到类似于下一张图中的内容。注意，编译器通常会将协程帧的地址保存在单独记录栈顶指针的寄存器中（MSVC将保存在rbp寄存器中）。

```
	栈                     寄存器               	堆
+----------------+ <-+
|  x()           |   |
| a  = 42        |   |                   +-->  +-----------+
| ret= f()+0x123 |   |    +------+       |     |  x()      |
+----------------+   +--- | rsp  |       |     | a =  42   |
|  f()           |        +------+       |     +-----------+
+----------------+        | rbp  | ------+
| ...            |        +------+
|                |
```

If the coroutine `x()` then calls another normal function `g()` it will look something like this.

如果协程 `x()` 之后又调用了另外一个普通函数 `g()`，它看起来将像下面一样。

```
	栈                     寄存器               	堆
+----------------+ <-+
|  g()           |   |
| ret= x()+0x45  |   |
+----------------+   |
|  x()           |   |
| coroframe      | --|-------------------+
| a  = 42        |   |                   +-->  +-----------+
| ret= f()+0x123 |   |    +------+             |  x()      |
+----------------+   +--- | rsp  |             | a =  42   |
|  f()           |        +------+             +-----------+
+----------------+        | rbp  |
| ...            |        +------+
|                |
```

When `g()` returns it will destroy its activation frame and restore `x()`’s activation frame. Let’s say we save `g()`’s return value in a local variable `b` which is stored in the coroutine frame.

当 `g()` 返回时，`g()` 的活动帧将被销毁，然后恢复 `x()` 的活动帧。假设我们将 `g()` 的返回值保存到一个存储在协程帧中的局部变量`b`  中。

```
	栈                     寄存器               	堆
+----------------+ <-+
|  x()           |   |
| a  = 42        |   |                   +-->  +-----------+
| ret= f()+0x123 |   |    +------+       |     |  x()      |
+----------------+   +--- | rsp  |       |     | a =  42   |
|  f()           |        +------+       |     | b = 789   |
+----------------+        | rbp  | ------+     +-----------+
| ...            |        +------+
|                |
```

If `x()` now hits a suspend-point and suspends execution without destroying its activation frame then execution returns to `f()`.

如果 `x()` 执行到了挂起点，它将挂起并且不去销毁自己的活动帧，然后将执行流交回 `f()`。

This results in the stack-frame part of `x()` being popped off the stack while leaving the coroutine-frame on the heap. When the coroutine suspends for the first time, a return-value is returned to the caller. This return value often holds a handle to the coroutine-frame that suspended that can be used to later resume it. When `x()` suspends it also stores the address of the resumption-point of `x()` in the coroutine frame (call it `RP` for resume-point).

这将导致 `x()` 的栈帧部分从栈中弹出，同时将协程帧保留在堆中。当协程第一次挂起时，一个返回值将被返回给调用者。这个返回值通常维护被挂起协程的协程帧的一个句柄，它可以被用来在之后恢复协程的执行。当 `x()` 被挂起时，它也会将 `x()` 的恢复点地址存储在协程帧中（RP表示恢复点）。

```
	栈                     寄存器               堆
                                        +----> +-----------+
                          +------+      |      |  x()      |
+----------------+ <----- | rsp  |      |      | a =  42   |
|  f()           |        +------+      |      | b = 789   |
| handle     ----|---+    | rbp  |      |      | RP=x()+99 |
| ...            |   |    +------+      |      +-----------+
|                |   |                  |
|                |   +------------------+
```

This handle may now be passed around as a normal value between functions. At some point later, potentially from a different call-stack or even on a different thread, something (say, `h()`) will decide to resume execution of that coroutine. For example, when an async I/O operation completes.

现在这个句柄可以作为一个正常的值在函数之间传递。之后的某个时间，协程可能会被不同的调用栈甚至不同的线程恢复执行。例如，当一个异步 I/O 操作完成时。

The function that resumes the coroutine calls a `void resume(handle)` function to resume execution of the coroutine. To the caller, this looks just like any other normal call to a `void`-returning function with a single argument.

恢复协程的函数调用一个 `void resume(handle)` 函数来恢复协程的执行。对于调用者而言，这和调用一个带有单个参数且没有返回值的普通函数没有什么区别。

This creates a new stack-frame that records the return-address of the caller to `resume()`, activates the coroutine-frame by loading its address into a register and resumes execution of `x()` at the resume-point stored in the coroutine-frame.

这将创建一个新的栈帧，它记录了调用者调用 `resume()` 时的地址 ，加载协程帧的地址到寄存器，找到存储在协程帧中的恢复点，然后在此恢复执行 `x()`。

```
	栈                     寄存器               	堆
+----------------+ <-+
|  x()           |   |                   +-->  +-----------+
| ret= h()+0x87  |   |    +------+       |     |  x()      |
+----------------+   +--- | rsp  |       |     | a =  42   |
|  h()           |        +------+       |     | b = 789   |
| handle         |        | rbp  | ------+     +-----------+
+----------------+        +------+
| ...            |
|                |
```

## In summary

I have described coroutines as being a generalisation of a function that has three additional operations - ‘Suspend’, ‘Resume’ and ‘Destroy’ - in addition to the ‘Call’ and ‘Return’ operations provided by “normal” functions.

我将协程看成函数的泛化，因为它除了拥有普通函数的 **Call** 和 **Return** 操作外，还拥有另外三个操作：**Suspend** 、**Resume** 和 **Destroy**。

I hope that this provides some useful mental framing for how to think of coroutines and their control-flow.

我希望这为如何思考协程及其控制流提供有用的思路。

In the next post I will go through the mechanics of the C++ Coroutines TS language extensions and explain how the compiler translates code that you write into coroutines.

在下一篇文章中，我将介绍C++ Cooutines TS语言扩展的机制，并解释编译器如何将您编写的代码转换为协程。

# C++ Coroutines: Understanding operator co_await

- 原文地址：https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await
- 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
- 本文永久链接：https://github.com/xitu/gold-miner/blob/master/TODO1/understanding-operator-co-await.m
- 译者：[7Ethan](https://github.com/7Ethan)

In the previous post on [Coroutine Theory](https://lewissbaker.github.io/2017/09/25/coroutine-theory) I described the high-level differences between functions and coroutines but without going into any detail on syntax and semantics of coroutines as described by the C++ Coroutines TS ([N4680](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf)).

在之前关于[协程理论](https://lewissbaker.github.io/2017/09/25/coroutine-theory)的博客中，我介绍了一些函数和协程在较高层次上的一些不同，但没有详细介绍 C++ 协程规范（N4680）中描述的语法和语义。

The key new facility that the Coroutines TS adds to the C++ language is the ability to suspend a coroutine, allowing it to be later resumed. The mechanism the TS provides for doing this is via the new `co_await` operator.

协程技术规范中，C++ 新增的关键新功能是能够挂起协程，并能够在之后恢复。技术规范为此提供的机制是通过新的 `co_await` 运算符去实现。

Understanding how the `co_await` operator works can help to demystify the behaviour of coroutines and how they are suspended and resumed. In this post I will be explaining the mechanics of the `co_await` operator and introduce the related **Awaitable** and **Awaiter** type concepts.

理解 `co_await` 运算符的工作原理可以帮助我们揭开协程行为的神秘面纱，并了解它们是如何被挂起和恢复的。在这篇文章中，我将解释 `co_await` 操作符的机制，并介绍 **Awaitable** 和 **Awaiter** 类型相关的概念。

But before I dive into `co_await` I want to give a brief overview of the Coroutines TS to provide some context.

在深入讲解 `co_await` 之前，我想简要介绍一下协程的技术规范，以提供一些背景知识。

## What does the Coroutines TS give us?

- Three new language keywords: `co_await`, `co_yield` and `co_return`
- Several new types in the `std::expermental` namespace:
  - `coroutine_handle<P>`
  - `coroutine_traits<Ts...>`
  - `suspend_always`
  - `suspend_never`
- A general mechanism that library writers can use to interact with coroutines and customise their behaviour.
- A language facility that makes writing asynchronous code a whole lot easier!

- 三个新的关键字：`co_await`， `co_yield`  和 `co_return`
- `std::experiment` 命名空间的几个新类型：
  - `coroutine_handle<P>`
  - `coroutine_traits<Ts...>`
  - `suspend_always`
  - `suspend_nerver`
- 一种能够让库的作者和协程交互并制定它们行为的通用机制。
- 一个使异步代码变得更加简单的语言工具。

The facilities the C++ Coroutines TS provides in the language can be thought of as a *low-level assembly-language* for coroutines. These facilities can be difficult to use directly in a safe way and are mainly intended to be used by library-writers to build higher-level abstractions that application developers can work with safely.

C++ 协程技术规范在语言中提供的工具，可以理解为协程的低级汇编语言。这些工具很难直接以安全的方式使用，主要是供库作者使用，用于构建应用程序开发人员可以安全使用的更高级别的抽象。

The plan is to deliver these new low-level facilities into an upcoming language standard (hopefully C++20) along with some accompanying higher-level types in the standard library that wrap these low-level building-blocks and make coroutines more accessible in a safe way for application developers.

未来会将这些新的低级工具交付给即将到来的语言标准（可能是 C++ 20），以及标准库中伴随的一些高级类型，这些高级类型封装了这些低级构建块，应用程序开发人员可以通过一种安全的方式轻松访问协程。

## Compiler <-> Library interaction（编译器和库的交互）

Interestingly, the Coroutines TS does not actually define the semantics of a coroutine. It does not define how to produce the value returned to the caller. It does not define what to do with the return value passed to the `co_return` statement or how to handle an exception that propagates out of the coroutine. It does not define what thread the coroutine should be resumed on.

有趣的是，协程技术规范实际上并没有定义协程的语义。他没有定义如何生成返回给调用者的值，没有定义如何处理传递给 `co_await` 语句的返回值，如何处理传递出协程的异常，它也没有定义应该恢复协程的线程。

Instead, it specifies a general mechanism for library code to customise the behaviour of the coroutine by implementing types that conform to a specific interface. The compiler then generates code that calls methods on instances of types provided by the library. This approach is similar to the way that a library-writer can customise the behaviour of a range-based for-loop by defining the `begin()`/`end()`

相反，它制定了库代码的通用机制，那就是通过实现符合特定接口类型来定制协程的行为。然后，编译器生成代码，在库提供的的类型实例上调用方法。这种方法类似于库作者通过定义 `begin()/end()` 方法或 `iterator` 类型来定制基于范围的for循环的实现。

The fact that the Coroutines TS doesn’t prescribe any particular semantics to the mechanics of a coroutine makes it a powerful tool. It allows library writers to define many different kinds of coroutines, for all sorts of different purposes.

协程技术规范没有对协程的机制规定特定的语义，这使它成为一个强大的工具。它允许库作者为各种不同的目的来定义许多不同类型的协程。

For example, you can define a coroutine that produces a single value asynchronously, or a coroutine that produces a sequence of values lazily, or a coroutine that simplifies control-flow for consuming `optional<T>` values by early-exiting if a `nullopt` value is encountered.

例如，你可以定义个异步生成单个值的协程，或者一个延迟生成一系列值的协程，或者如果遇到 `nullopt` 值，则通过提前退出来简化控制流以消耗 `optional<T>` 值的协程。

There are two kinds of interfaces that are defined by the coroutines TS: The **Promise** interface and the **Awaitable** interface.

协程技术规范定义了两种接口：**Promise** 接口和 **Awaitable** 接口。

The **Promise** interface specifies methods for customising the behaviour of the coroutine itself. The library-writer is able to customise what happens when the coroutine is called, what happens when the coroutine returns (either by normal means or via an unhandled exception) and customise the behaviour of any `co_await` or `co_yield` expression within the coroutine.

**Promise** 接口指定用于自定义协程本身行为的方法。库作者能够自定义调用协程时发生的事件，如协程返回时（通过正常方式或通过未处理的异常返回），或者自定义协程中任何 `co_await` 或 `co_yield` 表达式的行为。

The **Awaitable** interface specifies methods that control the semantics of a `co_await` expression. When a value is `co_await`ed, the code is translated into a series of calls to methods on the awaitable object that allow it to specify: whether to suspend the current coroutine, execute some logic after it has suspended to schedule the coroutine for later resumption, and execute some logic after the coroutine resumes to produce the result of the `co_await` expression.

**Awaitable** 接口指定控制 `co_await` 表达式语义的方法。当一个值被 `co_await`时，代码被转换为对 awaitable 对象上的方法的一系列调用。它可以指定：是否暂停当前协程，暂停调度协程以便稍后恢复后执行一些逻辑，还有在协程恢复后执行一些逻辑以产生 `co_await` 表达式的结果。

I’ll be covering details of the **Promise** interface in a future post, but for now let’s look at the **Awaitable** interface.

我将在以后的博客中介绍 **Promise** 接口的细节，现在我们先看看 **Awaitable** 接口。

## Awaiters and Awaitables: Explaining `operator co_await`

The `co_await` operator is a new unary operator that can be applied to a value. For example: `co_await someValue`.

`co_await` 运算符是一个新的一元运算符，可以应用于一个值。例如：`co_await someValue`。

The `co_await` operator can only be used within the context of a coroutine. This is somewhat of a tautology though, since any function body containing use of the `co_await` operator, by definition, will be compiled as a coroutine.

`co_await` 运算符只能在协程上下文中使用。这有点语义重复，因为根据定义，任何包含 `co_await` 运算符的函数体都将被编译为协程。

A type that supports the `co_await` operator is called an **Awaitable** type.

支持 `co_await` 运算符的类型称为 **Awaitable** 类型。

Note that whether or not the `co_await` operator can be applied to a type can depend on the context in which the `co_await` expression appears. The promise type used for a coroutine can alter the meaning of a `co_await` expression within the coroutine via its `await_transform` method (more on this later).

注意，`co_await` 运算符是否可以用做类型取决于 `co_await` 表达式出现的上下文。用于协程的 promise 类型可以通过其 `await_transform` 方法更改协程中 `co_await` 表达式的含义（稍后将详细介绍）。

To be more specific where required I like to use the term **Normally Awaitable** to describe a type that supports the `co_await` operator in a coroutine context whose promise type does not have an `await_transform` member. And I like to use the term **Contextually Awaitable** to describe a type that only supports the `co_await` operator in the context of certain types of coroutines due to the presence of an `await_transform` method in the coroutine’s promise type. (I’m open to better suggestions for these names here…)

为了在需要的地方更具体的表述，我喜欢使用术语 **Normally Awaitable** 来描述协程上下文中 `co_await` 运算符作用的类型的promise类型中没有 `await_transform` 成员。我喜欢使用术语 **Contextually Awaitable** 来描述一种类型，该类型仅在某些类型的协程中支持“ co_await”运算符，因为协程的promise类型中存在“ await_transform”方法。（我乐意接受其他更贴切的名字。。。）。

An **Awaiter** type is a type that implements the three special methods that are called as part of a `co_await` expression: `await_ready`, `await_suspend` and `await_resume`.

**Awaiter** 类型是一种实现了三个特殊方法的类型：`await_ready`， `await_suspend` 和 `await_resume`，它们是 `co_await` 表达式的一部分。

Note that I have shamelessly “borrowed” the term ‘Awaiter’ here from the C# `async` keyword’s mechanics that is implemented in terms of a `GetAwaiter()` method which returns an object with an interface that is eerily similar to the C++ concept of an **Awaiter**. See [this post](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-2-awaitable-awaiter-pattern) for more details on C# awaiters.

请注意，我从 C#  `async` 关键字的机制中“借用”了 “Awaiter” 这个术语，该机制是根据 `GetAwaiter()` 方法实现的，该方法返回一个对象，其接口与 C++ 的 Awaiter 概念惊人的相似。有关 C#  awaiters 的更多详细信息，请参考[这篇博文](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-2-awaitable-awaiter-pattern). 

Note that a type can be both an **Awaitable** type and an **Awaiter** type.

注意，一个类型可以同时是 **Awaitable** 类型和 **Awaiter** 类型。

When the compiler sees a `co_await <expr>` expression there are actually a number of possible things it could be translated to depending on the types involved.

当编译器遇到 `co_await <expr>`  表达式时，实际上可以根据所涉及的类型将其转换为许多可能的内容。

### Obtaining the Awaiter（获取 Awaiter）

The first thing the compiler does is generate code to obtain the **Awaiter** object for the awaited value. There are a number of steps to obtaining the awaiter object which are set out in N4680 section 5.3.8(3).

编译器做的第一件事是生成代码，以获取等待值的 **Awaiter** 对象。在 N4680 章节 5.3.8(3) 中，有很多步骤可以获得 awaiter。

Let’s assume that the promise object for the awaiting coroutine has type, `P`, and that `promise` is an l-value reference to the promise object for the current coroutine.

让我们假设等待协程的 promise 对象具有类型 `P`，并且 `promise`  是对当前协程的 promise 对象的 l-value 引用。

If the promise type, `P`, has a member named `await_transform` then `<expr>` is first passed into a call to `promise.await_transform(<expr>)` to obtain the **Awaitable** value, `awaitable`. Otherwise, if the promise type does not have an `await_transform` member then we use the result of evaluating `<expr>` directly as the **Awaitable** object, `awaitable`.

如果 promise 类型 `P` 有一个名为 `await_transform` 的成员，那么 `<expr>` 首先被传递给 `promise.await_transform(<expr>)` 以获得 **Awaitable** 的值。否则，如果 promise 类型没有 `await_transform`  成员，那么我们直接评估 `<expr>` 的结果作为 **Awaitable** 对象。

Then, if the **Awaitable** object, `awaitable`, has an applicable `operator co_await()` overload then this is called to obtain the **Awaiter** object. Otherwise the object, `awaitable`, is used as the awaiter object.

然后，如果 **Awaitable** 对象，有一个可用的运算符 `co_await()` 重载，那么调用它来获取 **Awaiter** 对象。否则，`awaitable` 的对象被用作 awaiter 的对象。

If we were to encode these rules into the functions `get_awaitable()` and `get_awaiter()`, they might look something like this:

如果我们将这些规则编码到 `get_awaitable()` 和 `get_awaiter()` 函数中，它们可能看起来像这样：

```c++
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
    if constexpr (has_any_await_transform_member_v<P>)
        return promise.await_transform(static_cast<T&&>(expr));
    else
        return static_cast<T&&>(expr)
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
    if constexpr (has_member_operator_co_await_v<Awaitable>)
        return static_cast<Awaitable&&>(awaitable).operator co_await();
    else if constexpr(has_non_member_operator_co_await_v<Awaitable&&>)
        return operator co_await(static_cast<Awaitable&&>(awaitable));
    else
        return static_cast<Awaitable&&>(awaitable);
}
```

### Awaiting the Awaiter（等待 Awaiter）

So, assuming we have encapsulated the logic for turning the `<expr>` result into an **Awaiter** object into the above functions then the semantics of `co_await <expr>` can be translated (roughly) as follows:

因此，假设我们已经将 `<expr>` 结果转换为 **Awaiter** 对象的逻辑封装了到上述函数中，那么 `co_await <expr>` 的语义可以（大致）这样转换：

```c++
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::experimental::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>)
    {
      awaiter.await_suspend(handle_t::from_promise(p));
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(p)))
      {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
```

The `void`-returning version of `await_suspend()` unconditionally transfers execution back to the caller/resumer of the coroutine when the call to `await_suspend()` returns, whereas the `bool`-returning version allows the awaiter object to conditionally resume the coroutine immediately without returning to the caller/resumer.

当 `await_suspend()` 的调用返回时，`await_suspend()` 的返回值为 `void` 的版本无条件地将执行转移回协程的调用者/恢复者，而返回值为 `bool` 的版本允许  awaiter 对象有条件地立即恢复协程，而不返回调用者/恢复者。

The `bool`-returning version of `await_suspend()` can be useful in cases where the awaiter might start an async operation that can sometimes complete synchronously. In the cases where it completes synchronously, the `await_suspend()` method can return `false` to indicate that the coroutine should be immediately resumed and continue execution.

`await_suspend()` 的 `bool` 返回版本在 awaiter 可能启动异步操作（有时可能同步完成）的情况下非常有用。在它同步完成的情况下，`await_suspend()` 方法可以返回 `false` 以指示应该立即恢复协程并继续执行。

At the `<suspend-coroutine>` point the compiler generates some code to save the current state of the coroutine and prepare it for resumption. This includes storing the location of the `<resume-point>` as well as spilling any values currently held in registers into the coroutine frame memory.

在 `<suspend-coroutine>` 处，编译器生成一些代码来保存协程的当前状态并准备恢复。这包括存储 `<resume-point>` 的断点位置，以及将当前保存在寄存器中的任何值到协程帧内存中。

The current coroutine is considered suspended after the `<suspend-coroutine>` operation completes. The first point at which you can observe the suspended coroutine is inside the call to `await_suspend()`. Once the coroutine is suspended it is then able to be resumed or destroyed.

在 `<suspend_coroutine>` 操作完成后，当前的协程被认为是暂停的。你可以观察到暂停的协程的第一个断点是在 `await_suspend()` 的调用中。协程暂停后，就可以恢复或销毁。

It is the responsibility of the `await_suspend()` method to schedule the coroutine for resumption (or destruction) at some point in the future once the operation has completed. Note that returning `false` from `await_suspend()` counts as scheduling the coroutine for immediate resumption on the current thread.

当操作完成后，`await_suspend()` 方法负责在将来的某个时刻调度并将协程恢复（或销毁）。注意，从 `await_suspend()` 返回 `false` 算作调度协程，以便在当前线程上立即恢复。

The purpose of the `await_ready()` method is to allow you to avoid the cost of the `<suspend-coroutine>` operation in cases where it is known that the operation will complete synchronously without needing to suspend.

`await_ready()` 方法的目的，是允许你在已知操作同步完成而不需要挂起的情况下避免 `<suspend-coroutine>` 操作的成本。

At the `<return-to-caller-or-resumer>` point execution is transferred back to the caller or resumer, popping the local stack frame but keeping the coroutine frame alive.

在 `<return-to-caller-or-resumer>` 断点处执行流转移回调用者或恢复者，弹出本地栈帧但是保持协程帧。

When (or if) the suspended coroutine is eventually resumed then the execution resumes at the `<resume-point>`. ie. immediately before the `await_resume()` method is called to obtain the result of the operation.

当（或者说如果）暂停的协程最终恢复时，执行将在 `<resume-point>` 断点处重新开始。即紧接在调用 `await_resume()` 方法获取操作结果之前。

The return-value of the `await_resume()` method call becomes the result of the `co_await` expression. The `await_resume()` method can also throw an exception in which case the exception propagates out of the `co_await` expression.

`await_resume()` 方法调用的返回值成为 `co_await` 表达式的结果。`await_resume()` 方法也可以抛出异常，在这种情况下异常从 `co_await` 表达式中抛出。

Note that if an exception propagates out of the `await_suspend()` call then the coroutine is automatically resumed and the exception propagates out of the `co_await` expression without calling `await_resume()`.

注意，如果异常从 `await_suspend()` 抛出，则协程会自动恢复，并且异常会从`co_await` 表达式抛出而不调用 `await_resume()`。

## Coroutine Handles

You may have noticed the use of the `coroutine_handle<P>` type that is passed to the `await_suspend()` call of a `co_await` expression.

你可能已经注意到 `coroutine_handle<P>` 类型的使用，该类型被传递给 `co_await` 表达式的 `await_suspend()` 调用。

This type represents a non-owning handle to the coroutine frame and can be used to resume execution of the coroutine or to destroy the coroutine frame. It can also be used to get access to the coroutine’s promise object.

该类型表示协程帧的非拥有句柄，可用于恢复协程的执行或销毁协程帧。它还可以用于访问协程的 promise 对象。

The `coroutine_handle` type has the following (abbreviated) interface:

`coroutine_handle` 类型具有以下接口：

```c++
namespace std::experimental
{
    template<typename Promise>
    struct coroutine_handle;
    
    template<>
    struct coroutine_handle<void>
    {
        bool done() const;
        void resume();
        void destroy();
        
        void* address() const;
        static coroutine_handle from_address(void* address);
    };
    
    template<typename Promise>
    struct coroutine_handle : coroutine_handle<void>
    {
      	Promise& promise() const;
        static coroutine_handle from_promise(Promise& promise);
        static coroutine_handle from_address(void* address);
    };
}
```

When implementing **Awaitable** types, they key method you’ll be using on `coroutine_handle` will be `.resume()`, which should be called when the operation has completed and you want to resume execution of the awaiting coroutine. Calling `.resume()` on a `coroutine_handle` reactivates a suspended coroutine at the `<resume-point>`. The call to `.resume()` will return when the coroutine next hits a `<return-to-caller-or-resumer>` point.

在实现 **Awaitable** 类型时，你将在 `coroutine_handle` 上使用的主要方法是 `.resume()`，当操作完成并希望恢复等待的协程的执行时，应该调用这个方法。在 `coroutine_handle` 上调用 `.resume()` 将在 `<resume-point>` 重新唤醒一个挂起的协程。当协程接下来遇到一个 `<return-to-caller-or-resumer>` 时，对 `.resume()` 的调用将返回。

The `.destroy()` method destroys the coroutine frame, calling the destructors of any in-scope variables and freeing memory used by the coroutine frame. You should generally not need to (and indeed should really avoid) calling `.destroy()` unless you are a library writer implementing the coroutine promise type. Normally, coroutine frames will be owned by some kind of RAII type returned from the call to the coroutine. So calling `.destroy()` without cooperation with the RAII object could lead to a double-destruction bug.

`.destroy()` 方法销毁协程帧，调用任何范围内变量的析构函数并释放协程帧使用的内存。通常，你不需要（实际上应该是避免）调用 `.destroy()` ，除非你是一个实现协程 promise 类型的库的编写者。通常，协程帧将由对协程的调用返回的某种 RAII（资源获取即初始化）类型拥有。所以在没有与 RAII 对象合作的情况下调用 `.destroy()` 可能会导致 double-destruction  的错误。

The `.promise()` method returns a reference to the coroutine’s promise object. However, like `.destroy()`, it is generally only useful if you are authoring coroutine promise types. You should consider the coroutine’s promise object as an internal implementation detail of the coroutine. For most **Normally Awaitable** types you should use `coroutine_handle<void>` as the parameter type to the `await_suspend()` method instead of `coroutine_handle<Promise>`.

`.promise()` 方法返回协程的 promise 对象的引用。但是，就像 `.destroy()` 一样，它通常只在你创建协程 promise 类型时才有用。你应该将协程的 promise 对象视为协程的内部实现细节。对于大多数 **Normally Awaitable** 类型，你应该使用 `coroutine_handle<void>` 作为 `await_suspend()` 方法的参数类型，而不是 `coroutine_handle<Promise>` 。

The `coroutine_handle<P>::from_promise(P& promise)` function allows reconstructing the coroutine handle from a reference to the coroutine’s promise object. Note that you must ensure that the type, `P`, exactly matches the concrete promise type used for the coroutine frame; attempting to construct a `coroutine_handle<Base>` when the concrete promise type is `Derived` can lead to undefined behaviour.

`coroutine_handle<P>::from_promise(P& promise)` 函数允许从对协程的 promise 对象的引用重构协程句柄。注意，你必须确保类型 `P` 与用于协程帧的具体 promise 类型完全匹配；当具体的 promise 类型是 `Derived` 时，试图通过 `coroutine_handle<Base>` 类型构造协程句柄将会出现未定义行为的错误。

The `.address()` / `from_address()` functions allow converting a coroutine handle to/from a `void*` pointer. This is primarily intended to allow passing as a ‘context’ parameter into existing C-style APIs, so you might find it useful in implementing **Awaitable** types in some circumstances. However, in most cases I’ve found it necessary to pass additional information through to callbacks in this ‘context’ parameter so I generally end up storing the `coroutine_handle` in a struct and passing a pointer to the struct in the ‘context’ parameter rather than using the `.address()` return-value.

`.address()` / `from_address()` 函数允许协程句柄和 `void*` 指针相互转化。这主要是为了允许作为 “context” 参数传递到现有的 C 风格 API 中，因此你可能会发现在某些情况下实现 **Awaitable** 类型很有用。但是，在大多数情况下，我发现有必要将附加信息传递给这个“context” 参数中的回调，因此我通常最终将 `coroutine_handle` 存储在结构中并将指针传递给 “context” 参数中的结构而不是使用 `.address()` 返回值。

## Synchronisation-free async code

One of the powerful design-features of the `co_await` operator is the ability to execute code after the coroutine has been suspended but before execution is returned to the caller/resumer.

`co_await` 运算符的一个强大的设计功能是在协程挂起后和执行权返回给执行者/恢复者之前执行代码的能力。

This allows an Awaiter object to initiate an async operation after the coroutine is already suspended, passing the `coroutine_handle` of the suspended coroutine to the operation which it can safely resume when the operation completes (potentially on another thread) without any additional synchronisation required.

这允许 Awaiter 对象在协程已经被挂起后发起异步操作，将被挂起的协程的句柄 `coroutine_handle` 传递给操作，当操作完成时（可能在另外一个线程上）它可以安全地恢复协程，而不需要额外的同步。

For example, by starting an async-read operation inside `await_suspend()` when the coroutine is already suspended means that we can just resume the coroutine when the operation completes without needing any thread-synchronisation to coordinate the thread that started the operation and the thread that completed the operation.

例如，当协程已经挂起时，在 `await_suspend()` 内启动异步读操作意味着我们可以在操作完成时恢复协程，而不需要任何线程同步来协调启动操作的线程和完成操作的线程。

```
Time     Thread 1                           Thread 2
  |      --------                           --------
  |      ....                               Call OS - Wait for I/O event
  |      Call await_ready()                    |
  |      <supend-point>                        |
  |      Call await_suspend(handle)            |
  |        Store handle in operation           |
  V        Start AsyncFileRead ---+            V
                                  +----->   <AsyncFileRead Completion Event>
                                            Load coroutine_handle from operation
                                            Call handle.resume()
                                              <resume-point>
                                              Call to await_resume()
                                              execution continues....
           Call to AsyncFileRead returns
         Call to await_suspend() returns
         <return-to-caller/resumer>
```

One thing to be *very* careful of when taking advantage of this approach is that as soon as you have started the operation which publishes the coroutine handle to other threads then another thread may resume the coroutine on another thread before `await_suspend()` returns and may continue executing concurrently with the rest of the `await_suspend()` method.

在利用这种方法是要特别注意的一件事情是，一旦开始将协程句柄发布到其他线程的操作，那么另外一个线程可以在 `await_suspend()` 返回之前恢复另一个线程上的协程，继续与 `await_suspend()` 方法的其余部分同时执行。

The first thing the coroutine will do when it resumes is call `await_resume()` to get the result and then often it will immediately destruct the **Awaiter** object (ie. the `this` pointer of the `await_suspend()` call). The coroutine could then potentially run to completion, destructing the coroutine and promise object, all before `await_suspend()` returns.

协程恢复时首先要做的事调用 `await_resume()` 来获取结果，然后常常会立即销毁 **Awaiter** 对象（即 调用`await_suspend()` 的 `this` 指针）。在 `await_suspend()` 返回之前，协程可能会运行完成，销毁协程和 promise 对象。

So within the `await_suspend()` method, once it’s possible for the coroutine to be resumed concurrently on another thread, you need to make sure that you avoid accessing `this` or the coroutine’s `.promise()` object because both could already be destroyed. In general, the only things that are safe to access after the operation is started and the coroutine is scheduled for resumption are local variables within `await_suspend()`.

所以在 `await_suspend()` 方法中，一旦协程有可能在另外的线程上同时恢复，你应该避免访问 `this` 指针或协程的 `.promise()` 对象，因为两者都已经可能被销毁了。一般来说，在启动操作并计划恢复协程之后，唯一可以安全访问的是 `await_suspend()` 中的局部变量。

### Comparison to Stackful Coroutines（和有栈协程比较）

I want to take a quick detour to compare this ability of the Coroutines TS stackless coroutines to execute logic after the coroutine is suspended with some existing common stackful coroutine facilities such as Win32 fibers or boost::context.

我想绕个弯，去比较一下协程技术规范中的无栈协程和一些现有的常见的有栈协程（如 Win32 纤程或 boost::context），在协程挂起后执行逻辑的能力。

With many of the stackful coroutine frameworks, the suspend operation of a coroutine is combined with the resumption of another coroutine into a ‘context-switch’ operation. With this ‘context-switch’ operation there is typically no opportunity to execute logic after suspending the current coroutine but before transferring execution to another coroutine.

对于许多有栈协程框架，一个协程的挂起操作与另一个协程的恢复操作相结合，形成一个“context-switch（上下文切换）”操作。使用这种“context-switch”操作，通常在当前协程挂起后，而在将执行流转移到另外一个协程之前，没有机会执行逻辑。

This means that if we want to implement a similar async-file-read operation on top of stackful coroutines then we have to start the operation *before* suspending the coroutine. It is therefore possible that the operation could complete on another thread before the coroutine is suspended and is eligible for resumption. This potential race between the operation completing on another thread and the coroutine suspending requires some kind of thread synchronisation to arbitrate and decide on the winner.

这意味着，如果我们想在有栈协程上实现类似的异步读取文件的操作，那么我们必须在挂起协程之前启动操作。因此，在协程暂停并有资格恢复前，有可能在另一个线程上完成该操作。在另一个线程上完成的操作和协程挂起之间的这种潜在竞争需要某种协程同步来仲裁，并决定胜利者。

There are probably ways around this by using a trampoline context that can start the operation on behalf of the initiating context after the initiating context has been suspended. However this would require extra infrastructure and an extra context-switch to make it work and it’s possible that the overhead this introduces would be greater than the cost of the synchronisation it’s trying to avoid.

通过使用 trampoline context 可以解决这个问题，该上下文可以在初始化上下文被挂起后代表初始上下文启动操作。然而，这将需要额外的基础设施和额外的上下文切换以使其工作，并且这里引入的开销可能大于它试图避免同步的成本。

## Avoiding memory allocations

Async operations often need to store some per-operation state that keeps track of the progress of the operation. This state typically needs to last for the duration of the operation and should only be freed once the operation has completed.

异步操作通常需要存储每个操作的一些状态，以跟踪操作的进度。这种状态通常需要在操作期间持续，并且只有在操作完成后才会释放。

For example, calling async Win32 I/O functions requires you to allocate and pass a pointer to an `OVERLAPPED` structure. The caller is responsible for ensuring this pointer remains valid until the operation completes.

例如，调用异步 Win32 I/O 函数需要你分配并传递指向 `OVERLAPPED` 结构的指针。调用者需要负责确保此指针有效，直到操作完成。

With traditional callback-based APIs this state would typically need to be allocated on the heap to ensure it has the appropriate lifetime. If you were performing many operations, you may need to allocate and free this state for each operation. If performance is an issue then a custom allocator may be used that allocates these state objects from a pool.

使用传统的基于回调的 API，通常需要在堆上分配此状态以确保其具有适当的生命周期。如果你执行了许多操作，则可能需要为每个操作分配并释放此状态。如果性能成了问题，那么可以使用自定义分配器从内存池中分配这些状态对象。

However, when we are using coroutines we can avoid the need to heap-allocate storage for the operation state by taking advantage of the fact that local variables within the coroutine frame will be kept alive while the coroutine is suspended.

然而，我们可以使用协程，通过利用协程帧中的局部变量在协程挂起后还回保持活跃的特性，避免为操作状态在堆上分配内存。

By placing the per-operation state in the **Awaiter** object we can effectively “borrow” memory from the coroutine frame for storing the per-operation state for the duration of the `co_await` expression. Once the operation completes, the coroutine is resumed and the **Awaiter** object is destroyed, freeing that memory in the coroutine frame for use by other local variables.

通过将每个操作状态放置在 **Awaiter** 对象中，我们可以从协程帧有效地 “borrow（借用）”存储器，用于在 `co_await` 表达式的持续时间内存储每个操作的状态。一旦操作完成，协程就会恢复并且销毁 **Awaiter** 对象，从而释放协程帧中的内存以供其他局部变量使用。

Ultimately, the coroutine frame may still be allocated on the heap. However, once allocated, a coroutine frame can be used to execute many asynchronous operations with only that single heap allocation.

最终，协程帧仍然是在堆上分配。然而，协程帧一旦分配了，就可以使用这个堆分配来执行许多异步操作。

If you think about it, the coroutine frame acts as a kind of really high-performance arena memory allocator. The compiler figures out at compile time the total arena size it needs for all local variables and is then able to allocate this memory out to local variables as required with zero overhead! Try beating that with a custom allocator ;)

如果你仔细想一想，就会发现协程帧就像一种高性能的 arena（竞技场）内存分配器。编译器在编译阶段就能计算出所有局部变量所有的 arena 总大小，然后根据需要将内存分配给局部变量，而开销为零！试着用自定义分配器打败它 :)。

## An example: Implementing a simple thread-synchronisation primitive（实现简单的线程同步原语）

Now that we’ve covered a lot of the mechanics of the `co_await` operator, I want to show how to put some of this knowledge into practice by implementing a basic awaitable synchronisation primitive: An asynchronous manual-reset event.

既然我们已经介绍了 `co_await` 运算符的许多机制，我想通过实现一个基本可等待同步原语来展示如何将这些只是付诸实践：异步手动重置事件。

The basic requirements of this event is that it needs to be **Awaitable** by multiple concurrently executing coroutines and when awaited needs to suspend the awaiting coroutine until some thread calls the `.set()` method, at which point any awaiting coroutines are resumed. If some thread has already called `.set()` then the coroutine should continue without suspending.

这个事件的基本要求是，他需要通过多个并发执行协程成为 **Awaitable** 状态，当等待时，需要挂起等待的协程，直到某个线程调用 `.set()` 方法，此时任何等待的协程都将恢复。如果某个线程已经调用了 `.set()` ，那么协程应该继续，而不是挂起。

Ideally we’d also like to make it `noexcept`, require no heap allocations and have a lock-free implementation.

理想情况下，我们还希望将其设置为 `noexcept` ，不需要在堆上分布，也不需要锁。

**Edit 2017/11/23: Added example usage for `async_manual_reset_event`**

**`2017/11/23更新：增加 async_manual_reset_event 示例`**

Example usage should look something like this:

示例用法如下：

```c++
T value;
async_manual_reset_event event;

// A single call to produce a value
void producer()
{
  value = some_long_running_computation();

  // Publish the value by setting the event.
  event.set();
}

// Supports multiple concurrent consumers
task<> consumer()
{
  // Wait until the event is signalled by call to event.set()
  // in the producer() function.
  co_await event;

  // Now it's safe to consume 'value'
  // This is guaranteed to 'happen after' assignment to 'value'
  std::cout << value << std::endl;
}
```

Let’s first think about the possible states this event can be in: ‘not set’ and ‘set’.

让我们首先考虑一下这个事件可能存在的状态： “not set” 和 “set”。

When it’s in the ‘not set’ state there is a (possibly empty) list of waiting coroutines that are waiting for it to become ‘set’.

当它处于 “not set” 状态时，有一队（可能为空）协程正在等待它变为 “set” 状态。

When it’s in the ‘set’ state there won’t be any waiting coroutines as coroutines that `co_await` the event in this state can continue without suspending.

当它处于 “set” 状态时，不会有任何等待的协程，因为在该状态下， `co_await` 的事件可以继续执行不用暂停。

This state can actually be represented in a single `std::atomic<void*>`.

- Reserve a special pointer value for the ‘set’ state. In this case we’ll use the `this` pointer of the event since we know that can’t be the same address as any of the list items.
- Otherwise the event is in the ‘not set’ state and the value is a pointer to the head of a singly linked-list of awaiting coroutine structures.

这个状态实际上可以用一个 `std::atomic<void*>` 来表示。

- 为 “set” 状态保留一个特殊的指针值。在这种情况下，我们将使用事件的 `this` 指针，因为我们知道该地址不会与任何列表项的地址相同。
- 否则，事件处于 “not set” 状态，并且该值是指向等待协程的单链表结构的头部指针。

We can avoid extra calls to allocate nodes for the linked-list on the heap by storing the nodes within an ‘awaiter’ object that is placed within the coroutine frame.

我们可以通过将节点存储在协程帧内的 “awaiter” 对象中，从而避免为链表在堆上分配节点引起的额外调用。

So let’s start with a class interface that looks something like this:

让我们从一个类接口开始，如下所示：

```c++
class async_manual_reset_event
{
public:

  async_manual_reset_event(bool initiallySet = false) noexcept;

  // No copying/moving
  async_manual_reset_event(const async_manual_reset_event&) = delete;
  async_manual_reset_event(async_manual_reset_event&&) = delete;
  async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
  async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;

  bool is_set() const noexcept;

  struct awaiter;
  awaiter operator co_await() const noexcept;

  void set() noexcept;
  void reset() noexcept;

private:

  friend struct awaiter;

  // - 'this' => set state
  // - otherwise => not set, head of linked list of awaiter*.
  mutable std::atomic<void*> m_state;

};
```

Here we have a fairly straight-forward and simple interface. The main thing to note at this point is that it has an `operator co_await()` method that returns an, as yet, undefined type, `awaiter`.

我们有一个相当直接和简单的接口。在这一点上，需要关注的是它有一个 `operator co_await()` 方法，它返回一个尚未定义的 `awaiter` 类型。

Let’s define the `awaiter` type now.

现在让我们来定义 `awaiter` 类型

### Defining the Awaiter

Firstly, it needs to know which `async_manual_reset_event` object it is going to be awaiting, so it will need a reference to the event and a constructor to initialise it.

首先，它需要知道它将等待哪一个 `async_manual_reset_event` 对象，因此它需要一个对 `async_manual_reset_event` 事件对象的引用，并且拥有相应的构造函数进行初始化。

It also needs to act as a node in a linked-list of `awaiter` values so it will need to hold a pointer to the next `awaiter` object in the list.

它还需要充当 `awaiter` 值链表中的节点，因此它需要持有指向列表中下一个 `awaiter` 对象的指针。

It also needs to store the `coroutine_handle` of the awaiting coroutine that is executing the `co_await` expression so that the event can resume the coroutine when it becomes ‘set’. We don’t care what the promise type of the coroutine is so we’ll just use a `coroutine_handle<>` (which is short-hand for `coroutine_handle<void>`).

它还需要存储正在执行 `co_await` 表达式的等待协程的 `coroutine_handle`，以便在事件变为 “set” 状态时事件可以恢复协程。我们不关心协程的 promise 类型是什么，所有我们只使用 `coroutine_handle<>` （这是 `coroutine_handle<void>` 的简写）。

Finally, it needs to implement the **Awaiter** interface, so it needs the three special methods: `await_ready`, `await_suspend` and `await_resume`. We don’t need to return a value from the `co_await` expression so `await_resume` can return `void`.

最后，它需要实现 **Awaiter** 接口，因此需要三种特殊的方法： `await_ready()`， `await_suspend`和`await_resume` 。我们不需要从 `co_await` 表达式返回一个值，因此 `await_resume` 可以返回 `void`。

Once we put all of that together, the basic class interface for `awaiter` looks like this:

当我们把所有这些都放到一起，`awaiter` 的基本接口如下所示：

```c++
struct async_manual_reset_event::awaiter
{
    awaiter(const async_manual_reset_event& event) noexcept
        :m_event(event)
    {}
    
    bool await_ready() const noexcept;
    bool await_suspend(std::experimental::coroutine_handle<> awaitingCoroutine) noexcept;
    void await_resume() noexcept {}
    
private:
    const async_manual_reset_event& m_event;
    std::experimental::coroutine_handle<> m_awaitingCoroutine;
    awaiter* m_next;
};
```

Now, when we `co_await` an event, we don’t want the awaiting coroutine to suspend if the event is already set. So we can define `await_ready()` to return `true` if the event is already set.

现在，当我们执行 `co_await` 一个事件时，如果事件已经设置，我们不希望等待协程暂停。因此，如果事件已经设置，我们可以定义 `await_ready()`  返回 `true`。

```c++
bool async_manual_reset_event::awaiter::await_ready() const noexcept
{
    return m_event.is_set();
}
```

Next, let’s look at the `await_suspend()` method. This is usually where most of the magic happens in an awaitable type.

接下来，让我们看一下 `await_suspend()` 方法。这通常是 awaitable 类型会发生莫名其妙的事情的地方。

First it will need to stash the coroutine handle of the awaiting coroutine into the `m_awaitingCoroutine` member so that the event can later call `.resume()` on it.

首先，它需要将等待协程的句柄存入 `m_awaitingCoroutine` 成员，以便事件稍后可以在其上调用 `.resume()` 。

Then once we’ve done that we need to try and atomically enqueue the awaiter onto the linked list of waiters. If we successfully enqueue it then we return `true` to indicate that we don’t want to resume the coroutine immediately, otherwise if we find that the event has concurrently been changed to the ‘set’ state then we return `false` to indicate that the coroutine should be resumed immediately.

然后，当我们完成了这一步，我们需要尝试将 awaiter 自动加入到 waiters 的链表中。如果我们成功的加入它，然后我们返回 `true` ，以表明我们不想立即恢复协程，否则，如果我们发现事件已并发地更改为 `set` 状态，那么我们返回 `false` ，以表明协程应立即恢复。

```c++
bool async_manual_reset_event::awaiter::await_suspend(
  std::experimental::coroutine_handle<> awaitingCoroutine) noexcept
{
  // Special m_state value that indicates the event is in the 'set' state.
  const void* const setState = &m_event;

  // Remember the handle of the awaiting coroutine.
  m_awaitingCoroutine = awaitingCoroutine;

  // Try to atomically push this awaiter onto the front of the list.
  void* oldValue = m_event.m_state.load(std::memory_order_acquire);
  do
  {
    // Resume immediately if already in 'set' state.
    if (oldValue == setState) return false; 

    // Update linked list to point at current head.
    m_next = static_cast<awaiter*>(oldValue);

    // Finally, try to swap the old list head, inserting this awaiter
    // as the new list head.
  } while (!m_event.m_state.compare_exchange_weak(
             oldValue,
             this,
             std::memory_order_release,
             std::memory_order_acquire));

  // Successfully enqueued. Remain suspended.
  return true;
}
```

Note that we use ‘acquire’ memory order when loading the old state so that if we read the special ‘set’ value then we have visibility of writes that occurred prior to the call to ‘set()’.

注意，在加载旧状态时，我们使用 “acquire” 内存顺序，如果我们读取特殊的 “set” 值时，那么我们看到就是在调用 “set()” 之前写入的值。

We require ‘release’ sematics if the compare-exchange succeeds so that a subsequent call to ‘set()’ will see our writes to m_awaitingCoroutine and prior writes to the coroutine state.

如果 compare-exchange 执行成功，我们需要 “release” 的状态，以便后续的 “set()” 调用将看到我们对 m_awaitingcorouting 的写入，以及之前对协程状态的写入。

### Filling out the rest of the event class

Now that we have defined the `awaiter` type, let’s go back and look at the implementation of the `async_manual_reset_event` methods.

现在我们已经定义了 `awaiter` 类型，让我们回过头来看看 `async_manual_reset_event` 方法的实现。

First, the constructor. It needs to initialise to either the ‘not set’ state with the empty list of waiters (ie. `nullptr`) or initialise to the ‘set’ state (ie. `this`).

首先是构造函数。它需要初始化为 “not set” 状态和空的 waiters 链表（即 `nullptr`）或初始化为 “set” 状态（即 `this`）。

```c++
async_manual_reset_event::async_manual_reset_event(bool initiallySet) noexcept
    :m_state(initially ? this : nullptr)
    {}
```

Next, the `is_set()` method is pretty straight-forward - it’s ‘set’ if it has the special value `this`:

接下来，`is_set()` 方法非常简单-如果它具有特殊值 `this` ，则为 “set”：

```c++
bool async_manual_reset_event::is_set() const noexcept
{
    return m_state.load(std::memory_order_acquire) == this;
}
```

Next, the `reset()` method. If it’s in the ‘set’ state we want to transition back to the empty-list ‘not set’ state, otherwise leave it as it is.

然后是 `reset()` 方法，如果它处于 “set” 状态，我们希望他转换为 “not set” 状态，否则保持原样。

```c++
void async_manual_reset_event::reset() noexcept
{
    void* oldValue = this;
    m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
}
```

With the `set()` method, we want to transition to the ‘set’ state by exchanging the current state with the special ‘set’ value, `this`, and then examine what the old value was. If there were any waiting coroutines then we want to resume each of them sequentially in turn before returning.

使用 `set()` 方法，我们希望通过使用特殊的 “set” 值（`this`）替换 “set” 状态，然后检查原来的值是什么。如果有任何等待的协程，那么我们希望在返回之前依次恢复它们。

```c++
void async_manual_reset_event::set() noexcept
{
  // Needs to be 'release' so that subsequent 'co_await' has
  // visibility of our prior writes.
  // Needs to be 'acquire' so that we have visibility of prior
  // writes by awaiting coroutines.
  void* oldValue = m_state.exchange(this, std::memory_order_acq_rel);
  if (oldValue != this)
  {
    // Wasn't already in 'set' state.
    // Treat old value as head of a linked-list of waiters
    // which we have now acquired and need to resume.
    auto* waiters = static_cast<awaiter*>(oldValue);
    while (waiters != nullptr)
    {
      // Read m_next before resuming the coroutine as resuming
      // the coroutine will likely destroy the awaiter object.
      auto* next = waiters->m_next;
      waiters->m_awaitingCoroutine.resume();
      waiters = next;
    }
  }
}
```

Finally, we need to implement the `operator co_await()` method. This just needs to construct an `awaiter` object.

最后，我们需要实现 `operator co_await()` 方法。这只需要构造一个 `awaiter` 对象。

```c++
async_manual_reset_event::awaiter
async_manual_reset_event::operator co_await() const noexcept
{
    return awaiter{*this};
}
```

我们终于完成它了，一个可等待的异步手动重置事件，具有无锁，无内存分配，noexcept实现。

If you want to have a play with the code or check out what it compiles down to under MSVC and Clang have a look at the [source on godbolt](https://godbolt.org/g/Ad47tH).

如果你想尝试一下代码，或者看看它编译到 MSVC 和 Clang 下面的代码，可以在 [godbolt](https://godbolt.org/g/Ad47tH)。

You can also find an implementation of this class available in the [cppcoro](https://github.com/lewissbaker/cppcoro) library, along with a number of other useful awaitable types such as `async_mutex` and `async_auto_reset_event`.

你还可以在 [cppcoro](https://github.com/lewissbaker/cppcoro) 库中找到此类的实现，以及许多其他有用的 awaitable 类型，例如 `async_mutex` 和 `async_auto_reset_event`。

## Closing Off（结束语）

This post has looked at how the `operator co_await` is implemented and defined in terms of the **Awaitable** and **Awaiter** concepts.

这篇文章介绍了如何根据 **Awaitable** 和 **Awaiter** 概念实现和定义运算符 `co_await`。

It has also walked through how to implement an awaitable async thread-synchronisation primitive that takes advantage of the fact that awaiter objects are allocated on the coroutine frame to avoid additional heap allocations.

它还介绍了如何实现一个等待的异步线程同步原语，该原语利用了协程帧分配 awaiter 对象的事实，以避免额外的堆分配。

I hope this post has helped to demystify the new `co_await` operator for you.

我希望这篇文章已经帮助你对 `co_await` 这个新的运算符有了更好的理解。

In the next post I’ll explore the **Promise** concept and how a coroutine-type author can customise the behaviour of their coroutine.

在下一篇博客中，我将讨论 **Promise** 概念以及协程类型作者如何定制其协程的行为。

## Thanks

I want to call out special thanks to Gor Nishanov for patiently and enthusiastically answering my many questions on coroutines over the last couple of years.

我要特别感谢 Gor Nishanov 在过去几年中耐心而热情地回答了我关于协程的许多问题。

And also to Eric Niebler for reviewing and providing feedback on an early draft of this post.

此外，还有 Eric Niebler 对本文的早期草稿进行审核并提供反馈

# C++ Coroutines: Understanding the promise type

原文地址：https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type

Sep 5, 2018

This post is the third in the series on the C++ Coroutines TS ([N4736](http://wg21.link/N4736)).

这是C++ Coroutine TS （N4736）系列文章的第三篇。

The previous articles in this series cover:

本系列的前几篇文章分别是：

- [Coroutine Theory](https://lewissbaker.github.io/2017/09/25/coroutine-theory)
- [Understanding operator co_await](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

In this post I look at the mechanics of how the compiler translates coroutine code that you write into compiled code and how you can customise the behaviour of a coroutine by defining your own **Promise** type.

在这篇文章中，我将探讨编译器是如何将你编写的协程代码转化成被编译的代码的，以及如何通过定义自己的 **Promise** 类型来定义协程的行为。

## Coroutine Concepts

The Coroutines TS adds three new keywords: `co_await`, `co_yield` and `co_return`. Whenever you use one of these coroutine keywords in the body of a function this triggers the compiler to compile this function as a coroutine rather than as a normal function.

Coroutine TS 中新添了三个关键字：`co_await`，`co_yield` 和 `co_return`。每当你在函数体中使用这些协程关键字时，都会触发编译器将该函数作为协程而不是普通函数进行编译。

The compiler applies some fairly mechanical transformations to the code that you write to turn it into a state-machine that allows it to suspend execution at particular points within the function and then later resume execution.

编译器会对你编写的代码就行一些相当机械的转换，以将其转换成状态机，从而使其可以在函数内特定地方挂起，然后在稍后恢复执行。

In the previous post I described the first of two new interfaces that the Coroutines TS introduces: The **Awaitable** interface. The second interface that the TS introduces that is important to this code transformation is the **Promise** interface.

在前一篇文章中，我介绍了 Coroutine TS 引入的两个新接口中的第一个：**Awaitable** 接口。TS 引入的第二个接口 **Promise** 接口，它对代码转换很重要。

The **Promise** interface specifies methods for customising the behaviour of the coroutine itself. The library-writer is able to customise what happens when the coroutine is called, what happens when the coroutine returns (either by normal means or via an unhandled exception) and customise the behaviour of any `co_await` or `co_yield` expression within the coroutine.

**Promise** 接口指定用于自定义协程自身行为的方法。库作者可以自定义调用协程时发生的事情，协程返回时（通过正常方式或者未处理的异常）发生的事情，并自定义协程中任何 `co_await` 或 `co_yield` 表达式的行为。

## Promise objects

The **Promise** object defines and controls the behaviour of the coroutine itself by implementing methods that are called at specific points during execution of the coroutine.

 **Promise** 对象通过实现协程执行期间在特定点调用的方法来定义和控制协程本身的行为。

*Before we go on, I want you to try and rid yourself of any preconceived notions of what a “promise” is. While, in some use-cases, the coroutine promise object does indeed act in a similar role to the* `std::promise` *part of a* `std::future` *pair, for other use-cases the analogy is somewhat stretched. It may be easier to think about the coroutine’s promise object as being a “coroutine state controller” object that controls the behaviour of the coroutine and can be used to track its state.*

在我们继续之前，我希望你先摆脱对 “promise” 的任何先入为主的观念。虽然在某些用例中，协程 promise 对象确实扮演者与 `std::future` 中的 `std::promise` 对象相似的角色，但是在其它用例中却不适用。将协程的 promise 对象看做是可以控制协程的行为并可用来追踪器状态的 “协程状态控制器” ，这可能更容易理解。

An instance of the promise object is constructed within the coroutine frame for each invocation of a coroutine function.

每次调用协程函数，都会在协程帧中构建一个 promise 对象的实例。

The compiler generates calls to certain methods on the promise object at key points during execution of the coroutine.

编译器在协程执行的关键点上，生成对 promise 对象特定方法的调用。

In the following examples, assume that the promise object created in the coroutine frame for a particular invocation of the coroutine is `promise`.

在以下实例中，假定协程帧中为特定协程调用创建的 promise 对象就是 `promise`。

When you write a coroutine function that has a body, `<body-statements>`, which contains one of the coroutine keywords (`co_return`, `co_await`, `co_yield`) then the body of the coroutine is transformed to something (roughly) like the following:

当你写的协程函数具有 <body-statements> 时，并且其中包含了协程关键字（`co_return`，`co_awiat`， `co_yield`）中的一个，那么协程体将被（大致）转化为以下内容：

```c++
{
    co_await promise.initial_suspend();
    try
    {
        <body_statements>
    }
    catch
    {
        promise.unhandled_exception();
    }
FinalSuspend:
    co_await promise.final_suspend();
}
```

When a coroutine function is called there are a number of steps that are performed prior to executing the code in the source of the coroutine body that are a little different to regular functions.

当调用协程函数时，在协程体中的代码执行之前执行了很多步骤，这些步骤和常规函数稍有不同。

Here is a summary of the steps (I’ll go into more detail on each of the steps below).

1. Allocate a coroutine frame using `operator new` (optional).
2. Copy any function parameters to the coroutine frame.
3. Call the constructor for the promise object of type, `P`.
4. Call the `promise.get_return_object()` method to obtain the result to return to the caller when the coroutine first suspends. Save the result as a local variable.
5. Call the `promise.initial_suspend()` method and `co_await` the result.
6. When the `co_await promise.initial_suspend()` expression resumes (either immediately or asynchronously), then the coroutine starts executing the coroutine body statements that you wrote.

这是步骤的摘要（我将在下面详细介绍每个步骤）。

1. 使用 `operator new` 分配协程帧（可选）
2. 将所有的函数参数复制到协程帧
3. 调用 promise 对象类型 `P` 的构造函数
4. 调用 `promise.get_return_object()` 方法获取结果，以便在协程首次挂起时将结果返回给调用方。将调用结果存为局部变量。
5. 调用 `promise.init_suspend()` 方法，然后 `co_await` 结果。
6. 当 `co_await promise.initial_suspend()` 表达式恢复时（立即或异步），协程将开始执行你编写的协程体语句。

Some additional steps are executed when execution reaches a `co_return` statement:

1. Call `promise.return_void()` or `promise.return_value(<expr>)`
2. Destroy all variables with automatic storage duration in reverse order they were created.
3. Call `promise.final_suspend()` and `co_await` the result.

当执行达到 `co_return` 语句时，将执行一些额外的步骤：

1. 调用 `promise.return_void()` 或 `promise.return_value(<expr>)`
2. 以相反的顺序销毁所有具有自动存储持续时间的变量
3. 调用 `promise.final_suspend()` ，然后 `co_await` 结果。

If instead, execution leaves `<body-statements>` due to an unhandled exception then:

1. Catch the exception and call `promise.unhandled_exception()` from within the catch-block.
2. Call `promise.final_suspend()` and `co_await` the result.

如果由于异常没有被处理而未执行 `<body-statements>` ，将执行：

1. 捕获异常并在 catch-block 中调用 `promise.unhandled_exception()`。
2. 调用 `promise.final_suspend()`，然后 `co_await` 结果。

Once execution propagates outside of the coroutine body then the coroutine frame is destroyed. Destroying the coroutine frame involves a number of steps:

1. Call the destructor of the promise object.
2. Call the destructors of the function parameter copies.
3. Call `operator delete` to free the memory used by the coroutine frame (optional)
4. Transfer execution back to the caller/resumer.

一旦协程执行完毕，协程帧将会被销毁。销毁协程帧涉及多个步骤：

1. 调用 promise 对象的析构函数
2. 调用函数参数副本的析构函数
3. 调用 `operator delete` 释放协程帧使用的内存（可选）
4. 将执行权转交给调用方/恢复方。

When execution first reaches a `<return-to-caller-or-resumer>` point inside a `co_await` expression, or if the coroutine runs to completion without hitting a `<return-to-caller-or-resumer>` point, then the coroutine is either suspended or destroyed and the return-object previously returned from the call to `promise.get_return_object()` is then returned to the caller of the coroutine.

当执行流首次到达 `co_await` 表达式内的某个 `<return-to-caller-resumer>` 点时，或者如果协程运行到完成而没有碰到 `<return-to-caller-or-resumer>` 点时，协程将会被挂起或者销毁，然后将先前通过调用 `promise.get_return_object()` 得到的对象返回给协程的调用方。

### Allocating a coroutine frame

First, the compiler generates a call to `operator new` to allocate memory for the coroutine frame.

首先，编译器会生成一个 `operator new` 调用，用来为协程帧分配内存。

If the promise type, `P`, defines a custom `operator new` method then that is called, otherwise the global `operator new` is called.

如果 promise 类型 `P` 自定义了 `operator new` 方法则调用这个方法，否则将调用全局的 `operator new`。

There are a few important things to note here:

这里有一些重要的事情值得注意：

The size passed to `operator new` is not `sizeof(P)` but is rather the size of the entire coroutine frame and is determined automatically by the compiler based on the number and sizes of parameters, size of the promise object, number and sizes of local variables and other compiler-specific storage needed for management of coroutine state.

传递给 `operator new`  的大小不是 `sizeof(P)`  的大小而是整个协程帧的大小，编译器会根据参数的数量和大小、promise 对象的大小、局部变量的数量和大小以及其他管理协程状态所必须的编译器存储自动决定大小。

The compiler is free to elide the call to `operator new` as an optimisation if:

- it is able to determine that the lifetime of the coroutine frame is strictly nested within the lifetime of the caller; and
- the compiler can see the size of coroutine frame required at the call-site.

在以下情况，编译器可以自由的取消对 `operator new` 的调用作为优化：

- 编译器可以确定协程帧的生命周期被严格的嵌套在调用者的生命周期内。
- 编译器可以在调用点得出所需的协程帧大小

In these cases, the compiler can allocate storage for the coroutine frame in the caller’s activation frame (either in the stack-frame or coroutine-frame part).

在这些情况下，编译器可以在调用者的活动帧中为协程分配协程帧（协程的栈帧或者协程帧）。

The Coroutines TS does not yet specify any situations in which the allocation elision is guaranteed, so you still need to write code as if the allocation of the coroutine frame may fail with `std::bad_alloc`. This also means that you usually shouldn’t declare a coroutine function as `noexcept` unless you are ok with `std::terminate()` being called if the coroutine fails to allocate memory for the coroutine frame.

Coroutines TS 尚未指定什么情况下分配是有保证的，所以，你仍然需要为协程帧分配失败时可能出现的 `std::bad_alloc` 错误编写处理代码。这也意味着，你不应该将协程函数声明为 `noexcept`，除非你可以保证会在协程帧分配失败时调用 `std::terminate`。

There is a fallback, however, that can be used in lieu of exceptions for handling failure to allocate the coroutine frame. This can be necessary when operating in environments where exceptions are not allowed, such as embedded environments or high-performance environments where the overhead of exceptions is not tolerated.

然而，有一种退路可以替代异常来处理分配协程帧失败的情况。这在异常被禁用的环境中，例如嵌入式环境或者不允许额外开销的高性能环境中，是必须被用到。

If the promise type provides a static `P::get_return_object_on_allocation_failure()` member function then the compiler will generate a call to the `operator new(size_t, nothrow_t)` overload instead. If that call returns `nullptr` then the coroutine will immediately call `P::get_return_object_on_allocation_failure()` and return the result to the caller of the coroutine instead of throwing an exception.

如果 promise 类型提供了静态成员函数 `P::get_return_object_on_allocation_failure()` ，则编译器将生成对重载的 `operator new(size_t, nothrow_t)` 的调用。如果该调用返回 `nullptr`，则协程会立即调用 `P::get_return_object_on_allocation_failure()` 并将结果返回给调用者，而不是抛出异常。

#### Customising coroutine frame memory allocation

Your promise type can define an overload of `operator new()` that will be called instead of global-scope `operator new` if the compiler needs to allocate memory for a coroutine frame that uses your promise type.

你可以为 promise 类型定义一个重载的 `operator new()`，这样，当编译器需要使用你的 promise 类型而去为协程帧分配内存时，调用的将是你定义的 `operator new()` ，而不是全局作用域中的。

For example:

```c++
struct my_promise_type
{
  void* operator new(std::size_t)
  {
      void* ptr = my_custom_allocate(size);
      if(!ptr)
          throw std::bad_alloc{};
      return ptr;
  }
  void operator delete(void* ptr, std::size_t size)
  {
      my_custom_free(ptr, size);
  }
  ...
};
```

“But what about custom allocators?”, I hear you asking.

“但是自定义分配器呢”，我听到你问。

You can also provide an overload of `P::operator new()` that takes additional arguments which will be called with lvalue references to the coroutine function parameters if a suitable overload can be found. This can be used to hook up `operator new` to call an `allocate()` method on an allocator that was passed as an argument to the coroutine function.

你还可以提供带有其他参数的 `P::operator new()` 的重载，如果可以找到合适的重载，则将使用对协程函数参数的左值引用来调用该重载。这可以用于 `operator new`  去调用作为参数传递给协程函数的分配器上 `allocate()` 方法。

You will need to do some extra work to make a copy of the allocator inside the allocated memory so you can reference it in the corresponding call to `operator delete` since the parameters are not passed to the corresponding `operator delete` call. This is because the parameters are stored in the coroutine-frame and so they will have already been destructed by the time that `operator delete` is called.

因为在调用相应的`operator delete` 的过程中没有传递参数，你需要做一些额外的工作来在分配的内存中保存 allocator 的副本，以便在相应的 `operator delete`调用中引用它。这是因为参数存储在协程帧中，因此在调用 `operator delete` 时它们已经被析构了。

For example, you can implement `operator new` so that it allocates extra space after the coroutine frame and use that space to stash a copy of the allocator that can be used to free the coroutine frame memory.

例如，你可以实现 `operator new` 以便它在协程帧之后分配额外的空间，并使用该空间存储可用于释放协程帧内存的 allocator 的副本。

For example:

```c++
template<typename ALLOCATOR>
struct my_promise_type
{
  template<typename... ARGS>
  void* operator new(std::size_t sz, std::allocator_arg_t, ALLOCATOR& allocator, ARGS&... args)
  {
    // Round up sz to next multiple of ALLOCATOR alignment
    std::size_t allocatorOffset =
      (sz + alignof(ALLOCATOR) - 1u) & ~(alignof(ALLOCATOR) - 1u);

    // Call onto allocator to allocate space for coroutine frame.
    void* ptr = allocator.allocate(allocatorOffset + sizeof(ALLOCATOR));

    // Take a copy of the allocator (assuming noexcept copy constructor here)
    new (((char*)ptr) + allocatorOffset) ALLOCATOR(allocator);

    return ptr;
  }

  void operator delete(void* ptr, std::size_t sz)
  {
    std::size_t allocatorOffset =
      (sz + alignof(ALLOCATOR) - 1u) & ~(alignof(ALLOCATOR) - 1u);

    ALLOCATOR& allocator = *reinterpret_cast<ALLOCATOR*>(
      ((char*)ptr) + allocatorOffset);

    // Move allocator to local variable first so it isn't freeing its
    // own memory from underneath itself.
    // Assuming allocator move-constructor is noexcept here.
    ALLOCATOR allocatorCopy = std::move(allocator);

    // But don't forget to destruct allocator object in coroutine frame
    allocator.~ALLOCATOR();

    // Finally, free the memory using the allocator.
    allocatorCopy.deallocate(ptr, allocatorOffset + sizeof(ALLOCATOR));
  }
}
```

To hook up the custom `my_promise_type` to be used for coroutines that pass `std::allocator_arg` as the first parameter, you need to specialise the `coroutine_traits` class (see section on `coroutine_traits` below for more details).

要将自定义的 `my_promise_type` 用作将 `std::allocator_arg` 作为第一个参数的协程，你需要对 `coroutine_traits` 类进行特化（更多详细信息，请见下面有关`coroutine_traits` 的部分）。

```c++
namespace std::experimental
{
    template<typename ALLOCATOR, typename... ARGS>
    struct coroutine_traits<my_return_type, std::allocator_arg_t, ALLOCATOR, ARGS...>
    {
        using promise_type = my_promise_type<ALLOCATOR>;
    }
}
```

Note that even if you customise the memory allocation strategy for a coroutine, **the compiler is still allowed to elide the call to your memory allocator**.

请注意，即使你为协程自定义了内存分配策略，**仍然允许编译器取消对你的内存分配器的调用**

### Copying parameters to the coroutine frame

The coroutine needs to copy any parameters passed to the coroutine function by the original caller into the coroutine frame so that they remain valid after the coroutine is suspended.

协程需要把原始调用者传递给协程函数的所有参数复制到协程帧中，以使它们在协程挂起后仍然有效。

If parameters are passed to the coroutine by value, then those parameters are copied to the coroutine frame by calling the type’s move-constructor.

如果参数按值传递给协程，则通过调用它们的移动构造函数将这些参数复制到协程帧中。

If parameters are passed to the coroutine by reference (either lvalue or rvalue), then only the references are copied into the coroutine frame, not the values they point to.

如果参数通过引用（左值或右值）传递给协程，则仅将引用复制到协程帧中，而不是复制它们指向的值。

Note that for types with trivial destructors, the compiler is free to elide the copy of the parameter if the parameter is never referenced after a reachable `<return-to-caller-or-resumer>` point in the coroutine.

注意，对于带有琐碎析构函数（trivial destructors）的类型，如果协程中的参数在一个可到达`<return-to-caller-or-resumer>` 点后不在被引用，则编译器可以自由的删除该参数的副本。

There are many gotchas involved when passing parameters by reference into coroutines as you cannot necessarily rely on the reference remaining valid for the lifetime of the coroutine. Many common techniques used with normal functions, such as perfect-forwarding and universal-references, can result in code that has undefined behaviour if used with coroutines. Toby Allsopp has written a [great article](https://toby-allsopp.github.io/2017/04/22/coroutines-reference-params.html) on this topic if you want more details.

当通过引用将参数传递给协程时，这里会有很多陷阱，因为你不能保证在协程的生命周期内引用都保持有效。如果与协程一起使用，普通函数使用的许多常见技术，例如，完美转发和万能引用，都可能导致代码具有未定义的行为。如果你想要了解更多详细信息，Toby Allsopp 撰写过一篇很棒的[文章](https://toby-allsopp.github.io/2017/04/22/coroutines-reference-params.html)。

If any of the parameter copy/move constructors throws an exception then any parameters already constructed are destructed, the coroutine frame is freed and the exception propagates back out to the caller.

如果任何参数的 copy/move 构造函数抛出异常，则会破坏所有已构造的参数，协程帧也将被释放，并将异常传播回调用者。

### Constructing the promise object

Once all of the parameters have been copied into the coroutine frame, the coroutine then constructs the promise object.

一旦所有的参数都复制到协程帧中，然后协程构造 promise 对象。

The reason the parameters are copied prior to the promise object being constructed is to allow the promise object to be given access to the post-copied parameters in its constructor.

在构造 promise 对象之前复制参数的原因是为了允许 promise 对象获得对后复制参数的访问权。

First, the compiler checks to see if there is an overload of the promise constructor that can accept lvalue references to each of the copied parameters. If the compiler finds such an overload then the compiler generates a call to that constructor overload. If it does not find such an overload then the compiler falls back to generating a call to the promise type’s default constructor.

首先，编译前先检查 promise 是否有接受每个复制参数的左值引用的构造函数重载。如果编译器发现了这样的重载，则编译器将生成对该构造函数重载的调用。如果没有找到，则编译器则会生成对 promise 类型的默认构造函数的调用。

Note that the ability for the promise constructor to “peek” at the parameters was a relatively recent change to the Coroutines TS, being adopted in [N4723](http://wg21.link/N4723) at the Jacksonville 2018 meeting. See [P0914R1](http://wg21.link/P0914R1) for the proposal. Thus it may not be supported by some older versions of Clang or MSVC.

注意，promise 构造函数能够“窥视（peek）”参数的功能是对 Coroutines TS 较新的改动，该更新在 2018 年 Jacksonville 会议的 [N4723](http://wg21.link/N4723) 中被采用。有关建议，请参阅 [P0914R1](http://wg21.link/P0914R1)。因此，某些较旧版本的 Clang 或 MSVC 可能不支持它。

If the promise constructor throws an exception then the parameter copies are destructed and the coroutine frame freed during stack unwinding before the exception propagates out to the caller.

如果 promise 构造函数抛出异常，在异常传播到调用者之前，栈展开期间将销毁参数副本并释放协程帧。

### Obtaining the return object

The first thing a coroutine does with the promise object is obtain the `return-object` by calling `promise.get_return_object()`.

协程对 promise 对象所做的第一件事是通过调用 `promise.get_return_object()` 获得 `return-object`。

The `return-object` is the value that is returned to the caller of the coroutine function when the coroutine first suspends or after it runs to completion and execution returns to the caller.

`return-object` 是在协程首次挂起或运行完成时协程函数返回的值，然后执行权交还给调用者。

You can think of the control flow going something (very roughly) like this:

你可以想象控制流程（大致）如下：

```c++
// Pretend there's a compiler-generated structure called 'coroutine_frame'
// that holds all of the state needed for the coroutine. It's constructor
// takes a copy of parameters and default-constructs a promise object.
struct coroutine_frame { ... };

T some_coroutine(P param)
{
  auto* f = new coroutine_frame(std::forward<P>(param));

  auto returnObject = f->promise.get_return_object();

  // Start execution of the coroutine body by resuming it.
  // This call will return when the coroutine gets to the first
  // suspend-point or when the coroutine runs to completion.
  coroutine_handle<decltype(f->promise)>::from_promise(f->promise).resume();

  // Then the return object is returned to the caller.
  return returnObject;
}
```

Note that we need to obtain the return-object before starting the coroutine body since the coroutine frame (and thus the promise object) may be destroyed prior to the call to `coroutine_handle::resume()` returning, either on this thread or possibly on another thread, and so it would be unsafe to call `get_return_object()` after starting execution of the coroutine body.

注意，我们需要在启动协程体之前获取 `return-object`，因为协程帧（以及对应的 promise 对象）可能在调用 `coroutine_handle::resume()` 返回之前在此线程或者可能在另一个线程上被销毁，因此在执行协程体之后调用 `get_return_object` 是不安全的。

### The initial-suspend point

The next thing the coroutine executes once the coroutine frame has been initialised and the return object has been obtained is execute the statement `co_await promise.initial_suspend();`.

一旦初始化了协程帧并获得了返回对象，协程将要执行的下一件事是执行 `co_await promise.initial_suspend();` 语句。

This allows the author of the `promise_type` to control whether the coroutine should suspend before executing the coroutine body that appears in the source code or start executing the coroutine body immediately.

这使得 `promise_type` 的作者可以控制协程在执行协程体中的代码前，是挂起还是立即执行协程体。

If the coroutine suspends at the initial suspend point then it can be later resumed or destroyed at a time of your choosing by calling `resume()` or `destroy()` on the coroutine’s `coroutine_handle`.

如果协程在初始挂起点挂起，则可以在稍后你选择的时间通过在协程的 `coroutine_handle` 上调用 `resume()` 或 `destroy()` 恢复或销毁协程。 

The result of the `co_await promise.initial_suspend()` expression is discarded so implementations should generally return `void` from the `await_resume()` method of the awaiter.

`co_await promise.initial_suspend()` 表达式的结果将被丢弃，因此 awaiter 的`await_resume()` 实现通常返回 `void`。

It is important to note that this statement exists outside of the `try`/`catch` block that guards the rest of the coroutine (scroll back up to the definition of the coroutine body if you’ve forgotten what it looks like). This means that any exception thrown from the `co_await promise.initial_suspend()` evaluation prior to hitting its `<return-to-caller-or-resumer>` will be thrown back to the caller of the coroutine after destroying the coroutine frame and the return object.

需要非常注意的是，该语句位于 `try/catch` 语句块外部，`try/catch` 语句块保护协程的其余部分（如果忘记了具体的实现，请返回协程体的定义）。这意味着，在到达 `<return-to-caller-or-reusmer>` 之前， `co_await promise.initial_suspend()` 求值中抛出的异常将在协程帧和返回对象被销毁后返回给协程的调用者。

Be aware of this if your `return-object` has RAII semantics that destroy the coroutine frame on destruction. If this is the case then you want to make sure that `co_await promise.initial_suspend()` is `noexcept` to avoid double-free of the coroutine frame.

注意一点，如果返回对象具有RAII语义，它会在析构函数中销毁协程帧。如果是这种情况，你需要保证 `co_await promise.initial_suspend()` 是 `noexcept`，以避免 double-free 协程帧。

*Note that there is a proposal to tweak the semantics so that either all or part of the* `co_await promise.initial_suspend()` *expression lies inside try/catch block of the coroutine-body so the exact semantics here are likely to change before coroutines are finalised.*

*注意，有人建议调整语义，以使 `co_await promise.initial_suspend()` 表达式的全部或者部分位于协程体的 try/catch 块内，因此在确定协程之前，此处的确切语义可能会发生变化*

For many types of coroutine, the `initial_suspend()` method either returns `std::experimental::suspend_always` (if the operation is lazily started) or `std::experimental::suspend_never` (if the operation is eagerly started) which are both `noexcept` awaitables so this is usually not an issue.

对于许多类型的协程，`initial_suspend()` 方法要么返回 `std::experimental::suspend_always` （如果该操作是延迟启动的），要么返回的是 `std::experimental::suspend_never` （如果该操作是立即执行的），它们都是 `noexcept` 的 awaitables ，因此通常这不是问题。

### Returning to the caller

When the coroutine function reaches its first `<return-to-caller-or-resumer>` point (or if no such point is reached then when execution of the coroutine runs to completion) then the `return-object` returned from the `get_return_object()` call is returned to the caller of the coroutine.

当协程函数首次达到 `<return-to-caller-resumer>` 点（或者如果没有这样的暂停点，则协程运行至完成），然后将从 `get_return_object()` 调用返回的 `return-object` 返回给协程的调用者。

Note that the type of the `return-object` doesn’t need to be the same type as the return-type of the coroutine function. An implicit conversion from the `return-object` to the return-type of the coroutine is performed if necessary.

注意，`return-object` 的类型不必与协程函数的返回类型相同。如果需要，可以执行 `return-object` 到协程返回类型的隐式转换。

*Note that Clang’s implementation of coroutines (as of 5.0) defers executing this conversion until the return-object is returned from the coroutine call, whereas MSVC’s implementation as of 2017 Update 3 performs the conversion immediately after calling* `get_return_object()`*. Although the Coroutines TS is not explicit on the intended behaviour, I believe MSVC has plans to change their implementation to behave more like Clang’s as this enables some* [interesting use cases](https://github.com/toby-allsopp/coroutine_monad)*.*

*注意，Clang的协程实现（自 5.0 开始）推迟执行此转换，直到从协程调用返回 return-object，而 2017 年 Update 3 的 MSVC 的实现则在调用 `get_return_object` 之后立即执行转换。尽管 Coroutines TS 没有明确指出预期的行为，但我相信 MSCV 计划将其实现更改为更类似 Clang 的行为，因为这可以实现一些 [有趣的用例](https://github.com/toby-allsopp/coroutine_monad)*

### Returning from the coroutine using `co_return`

When the coroutine reaches a `co_return` statement, it is translated into either a call to `promise.return_void()` or `promise.return_value(<expr>)` followed by a `goto FinalSuspend;`.

当协程遇到 `co_return` 语句时，它将被转换成  `promise.return_void()` 或 `promise.reutrn_value(<expr>)` 调用，然后后面紧跟一个 `goto FinalSuspend;` 。

The rules for the translation are as follows:

转换规则如下：

- `co_return;`
  -> `promise.return_void();`
- `co_return <expr>;`
  -> `<expr>; promise.return_void();` if `<expr>` has type `void`
  -> `promise.return_value(<expr>);` if `<expr>` does not have type `void`

The subsequent `goto FinalSuspend;` causes all local variables with automatic storage duration to be destructed in reverse order of construction before then evaluating `co_await promise.final_suspend();`.

随后的 `goto FinalSuspend;` 操作将引起所有具有自动存储持续时间的局部变量以相反的构造顺序被析构，然后对 `co_await promise.final_suspend();` 求值。

Note that if execution runs off the end of a coroutine without a `co_return` statement then this is equivalent to having a `co_return;` at the end of the function body. In this case, if the `promise_type` does not have a `return_void()` method then the behaviour is undefined.

注意，如果在没有遇到 `co_return` 语句的情况下执行完整个协程，相当于在函数体末尾执行了 `co_return;` 语句。如果在 `promise_type` 没有 `return_void` 方法的情况下，则行为是未定义的。

If either the evaluation of `<expr>` or the call to `promise.return_void()` or `promise.return_value()` throws an exception then the exception still propagates to `promise.unhandled_exception()` (see below).

如果在对 `<expr>` 求值或调用 `promise.return_void` 或 `promise.return_value()` 抛出了异常，该异常会传播到 `promise.unhandled_exception()` （见下文）

### Handling exceptions that propagate out of the coroutine body

If an exception propagates out of the coroutine body then the exception is caught and the `promise.unhandled_exception()` method is called inside the `catch` block.

如果异常从协程体内传播出去，则异常会在 `catch` 块中被捕获并调用 `promise.unhandled_exception()` 方法。

Implementations of this method typically call `std::current_exception()` to capture a copy of the exception to store it away to be later rethrown in a different context.

此方法通常调用 `std::current_exception()` 捕获储存异常的副本，随后在不同上下文中重新抛出。

Alternatively, the implementation could immediately rethrow the exception by executing a `throw;` statement. For example see [folly::Optional](https://github.com/facebook/folly/blob/4af3040b4c2192818a413bad35f7a6cc5846ed0b/folly/Optional.h#L587) However, doing so will (likely - see below) cause the the coroutine frame to be immediately destroyed and for the exception to propagate out to the caller/resumer. This could cause problems for some abstractions that assume/require the call to `coroutine_handle::resume()` to be `noexcept`, so you should generally only use this approach when you have full control over who/what calls `resume()`.

或者，该实现也可以通过立即执行 `throw;` 语句重新抛出异常。如同 [folly::Optional](https://github.com/facebook/folly/blob/4af3040b4c2192818a413bad35f7a6cc5846ed0b/folly/Optional.h#L587) 例子，然后，一旦这样做（或者类似--参见下文）将会导致协程帧被立即析构并将异常扩散到调用者/恢复者。这可能会给某些假设/要求 `coroutine_handle::resume()` 的调用是 `noexcept` 的抽象造成问题，因此通常只应在完全控制谁或什么调用 `resume()` 时使用此方法。

Note that the current [Coroutines TS](http://wg21.link/N4736) wording is a [little unclear](https://github.com/GorNishanov/CoroutineWording/issues/17) on the intended behaviour if the call to `unhandled_exception()` rethrows the exception (or for that matter if any of the logic outside of the try-block throws an exception).

注意，如果调用 `unhandled_exception()` 重新抛出异常（或者 try-block 外的逻辑代码抛出异常），现在的 Coroutines TS 并没有给出明确预期的行为。

My current interpretation of the wording is that if control exits the coroutine-body, either via exception propagating out of `co_await promise.initial_suspend()`, `promise.unhandled_exception()` or `co_await promise.final_suspend()` or by the coroutine running to completion by `co_await p.final_suspend()` completing synchronously then the coroutine frame is automatically destroyed before execution returns to the caller/resumer. However, this interpretation has its own issues.

我目前对该用语的解释是，如果控制离开协程体，无论是通过 `co_await promise.initial_suspend()`， `promise.unhandled_exception()` 或 `co_await promise.final_suspend()` 扩散出来的异常，还是同步运行到协程完成的 `co_await p.final_suspend()` ，则协程帧会在执行流转回调用者/恢复者之前自动销毁。但是，这种解释有其自身的问题。

A future version of the Coroutines specification will hopefully clarify the situation. However, until then I’d stay away from throwing exceptions out of `initial_suspend()`, `final_suspend()` or `unhandled_exception()`. Stay tuned!

未来版本的 Coroutines TS 有希望明确这种情况。但是，在那之前，我将避免从 `initial_suspend()`，`final_suspend()` 和 `unhandled_exception()` 中抛出异常。敬请关注。

### The final-suspend point

Once execution exits the user-defined part of the coroutine body and the result has been captured via a call to `return_void()`, `return_value()` or `unhandled_exception()` and any local variables have been destructed, the coroutine has an opportunity to execute some additional logic before execution is returned back to the caller/resumer.

一旦执行流离开协程体中用户自定义的部分，并通过 `return_void()`，`return_value()` 或 `unhandled_exception()` 调用获得结果，同时，所有的局部变量将会被销毁，协程在将执行权返回给调用者/恢复者之前有机会执行一些额外的逻辑。

The coroutine executes the `co_await promise.final_suspend();` statement.

协程执行 `co_await promise.final_suspend();` 语句。

This allows the coroutine to execute some logic, such as publishing a result, signalling completion or resuming a continuation. It also allows the coroutine to optionally suspend immediately before execution of the coroutine runs to completion and the coroutine frame is destroyed.

这允许协程执行一些逻辑，比如发布一个结果，通过信号完成或回复一个协程。也允许协程可选地在协程执行完并在销毁协程帧之前立即挂起。

Note that it is undefined behaviour to `resume()` a coroutine that is suspended at the `final_suspend` point. The only thing you can do with a coroutine suspended here is `destroy()` it.

注意，`resume()` 在 `final_suspend` 点挂起的协程，行为是未定义的。协程在这点挂起时，你唯一能做的就是 `destroy()` 它。

The rationale for this limitation, according to Gor Nishanov, is that this provides several optimisation opportunities for the compiler due to the reduction in the number of suspend states that need to be represented by the coroutine and a potential reduction in the number of branches required.

根据 Gor Nishanov 的说话，做这样限制的原理是，可以减少协程挂起时所需的状态的数量和潜在的所需的分支的数量，这为编译器提供了一些优化机会。

Note that while it is allowed to have a coroutine not suspend at the `final_suspend` point, **it is recommended that you structure your coroutines so that they do suspend at `final_suspend`** where possible. This is because this forces you to call `.destroy()` on the coroutine from outside of the coroutine (typically from some RAII object destructor) and this makes it much easier for the compiler to determine when the scope of the lifetime of the coroutine-frame is nested inside the caller. This in turn makes it much more likely that the compiler can elide the memory allocation of the coroutine frame.

注意，虽然允许协程可以不在 `final_suspend` 点挂起，但是还是建议你尽可能的构建在 `final_suspend()` 挂起的协程。因为这会强制你在协程外部调用协程的 `.destroy()`（通常是从某些 RAII 对象的析构函数中），这使得编译器更容易确定协程帧的生命周期嵌套在调用者的作用域的生命周期中。 这样做更有可能让编译器取消协程帧的内存分配。

### How the compiler chooses the promise type

So lets look now at how the compiler determines what type of promise object to use for a given coroutine.

现在让我们看一看编译器是如何确定协程使用的 promise 的类型的。

The type of the promise object is determined from the signature of the coroutine by using the `std::experimental::coroutine_traits` class.

promise 的类型是通过协程的签名确定的，而协程签名是使用 `std::experimental::coroutine_traits` 类确定的。

If you have a coroutine function with signature:

如果你有一个协程函数签名：

```c++
task<float> foo(std::string x, bool flag);
```

Then the compiler will deduce the type of the coroutine’s promise by passing the return-type and parameter types as template arguments to `coroutine_traits`.

编译器将会根据传给 `coroutine_traits` 模板的返回类型和参数类型来推断协程的 promise 类型。

```c++
typename coroutine_traits<task<float>, std::string, bool>::promise_type;
```

If the function is a non-static member function then the class type is passed as the second template parameter to `coroutine_traits`. Note that if your method is overloaded for rvalue-references then the second template parameter will be an rvalue reference.

如果是一个非静态成员函数，则将类类型作为第二个模板参数传给 `coroutine_traits`。注意，如果你的方法重载了右值引用，那么第二个模板参数将是右值引用。

For example, if you have the following methods:

例如，如果你有下面的这些方法：

```c++
task<void> my_class::method1(int x) const;
task<foo> my_class::method2() &&;
```

编译器将使用下面的 promise 类型：

```c++
// method1 promise type
typename coroutine_traits<task<void>, const my_class&, int>::promise_type;

// method2 promise type
typename coroutine_traits<task<foo>, my_class&&>::promise_type;
```

The default definition of `coroutine_traits` template defines the `promise_type` by looking for a nested `promise_type` typedef defined on the return-type. ie. Something like this (but with some extra SFINAE magic so that `promise_type` is not defined if `RET::promise_type` is not defined).

`coroutine_traits` 模板中默认的 `promise_type` 定义，是去查找嵌套定义在 return-type 中的 `promise_type`。类似这样（但由于带有一些额外的 SFINAE魔法，如果 `RET::promise_type` 如果未定义，则 `promise_type` 也是未定义的）。

```c++
namespace std::experimental
{
    template<typename RET, typename... ARGS>
    struct coroutine_traits<RET, ARGS...>
    {
        using promise_type = typename RET::promise_type;
    };
}
```

So for coroutine return-types that you have control over, you can just define a nested `promise_type` in your class to have the compiler use that type as the type of the promise object for coroutines that return your class.

因此，你可以这样控制协程的返回类型，只需在你的类中定义一个嵌套的 `promise_type`， 编译器将会使用该类型作为协程的 promise 对象的类型。

For example:

```c++
template<typename T>
struct task
{
  using promise_type = task_promise<T>;  
};
```

However, for coroutine return-types that you don’t have control over you can specialise the `coroutine_traits` to define the promise type to use without needing to modify the type.

然而，对于你无法控制的协程返回类型，你可以特化 `coroutine_traits` 来定义要适用的 promise 类型，而无需修改类型。

For example, to define the promise-type to use for a coroutine that returns `std::optional<T>`:

例如，定义 promise 类型是  `std::optional<T>` 的协程：

```c++
namespace std::experimental
{
    template<typename T, typename... ARGS>
    struct coroutine_traits<std::optional<T>, ARGS...>
    {
        using promise_type = optional_promise<T>;
    };
}
```

### Identifying a specific coroutine activation frame

When you call a coroutine function, a coroutine frame is created. In order to resume the associated coroutine or destroy the coroutine frame you need some way to identify or refer to that particular coroutine frame.

调用协程函数时，协程帧将被创建。为了恢复关联的协程或者销毁协程帧，你需要某些方式去标识或者引用特定的协程帧。

The mechanism the Coroutines TS provides for this is the `coroutine_handle` type.

Coroutines TS 为此提供的机制是 `coroutine_handle` 类型。

The (abbreviated) interface of this type is as follows:

此类型的接口如下（简略）：

```c++
namespace std::experimental
{
  template<typename Promise = void>
  struct coroutine_handle;

  // Type-erased coroutine handle. Can refer to any kind of coroutine.
  // Doesn't allow access to the promise object.
  template<>
  struct coroutine_handle<void>
  {
    // Constructs to the null handle.
    constexpr coroutine_handle();

    // Convert to/from a void* for passing into C-style interop functions.
    constexpr void* address() const noexcept;
    static constexpr coroutine_handle from_address(void* addr);

    // Query if the handle is non-null.
    constexpr explicit operator bool() const noexcept;

    // Query if the coroutine is suspended at the final_suspend point.
    // Undefined behaviour if coroutine is not currently suspended.
    bool done() const;

    // Resume/Destroy the suspended coroutine
    void resume();
    void destroy();
  };

  // Coroutine handle for coroutines with a known promise type.
  // Template argument must exactly match coroutine's promise type.
  template<typename Promise>
  struct coroutine_handle : coroutine_handle<>
  {
    using coroutine_handle<>::coroutine_handle;

    static constexpr coroutine_handle from_address(void* addr);

    // Access to the coroutine's promise object.
    Promise& promise() const;

    // You can reconstruct the coroutine handle from the promise object.
    static coroutine_handle from_promise(Promise& promise);
  };
}
```

You can obtain a `coroutine_handle` for a coroutine in two ways:

1. It is passed to the `await_suspend()` method during a `co_await` expression.
2. If you have a reference to the coroutine’s promise object, you can reconstruct its `coroutine_handle` using `coroutine_handle<Promise>::from_promise()`.

你可以通过两种方式获取协程的 `coroutine_handle`：

1. 在 `co_await` 表达式期间，将它传递给 `await_suspend()` 方法
2. 如果你有一个协程 promise 对象的引用，你可以通过 使用`coroutine_handle<Promise>::from_promise()` 重构 `coroutine_handle`。

The `coroutine_handle` of the awaiting coroutine will be passed into the `await_suspend()` method of the awaiter after the coroutine has suspended at the `<suspend-point>` of a `co_await` expression. You can think of this `coroutine_handle` as representing the continuation of the coroutine in a [continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style) call.

当协程在 `co_await` 表达式的 `<suspend-point>` 被挂起后，等待协程的 `coroutine_handle` 会被传递给 awaiter 的 `await_suspend()` 方法。你可以将 `coroutine_handle` 视为 [continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style) 调用中延续了协程。

Note that the `coroutine_handle` is **NOT** and RAII object. You must manually call `.destroy()` to destroy the coroutine frame and free its resources. Think of it as the equivalent of a `void*` used to manage memory. This is for performance reasons: making it an RAII object would add additional overhead to coroutine, such as the need for reference counting.

注意，`coroutine_handle` 并不是 RAII 对象。你必须手动调用 `.destroy()` 去销毁协程帧并释放资源。你可以认为它干了和用 `void*` 管理内存一样的事。这是出于性能方面的考虑：把它变成 RAII 对象会增加协程的额外开销，例如需要引用计数。

You should generally try to use higher-level types that provide the RAII semantics for coroutines, such as those provided by [cppcoro](https://github.com/lewissbaker/cppcoro) (shameless plug), or write your own higher-level types that encapsulate the lifetime of the coroutine frame for your coroutine type.

通常，你应该尝试为协程提供拥有 RAII 语义的高级类型，例如 [cppcoro](https://github.com/lewissbaker/cppcoro) （）提供的那些类型，或者为你的协程类型编写高级类型来封装协程帧的生命周期。

### Customising the behaviour of `co_await`

The promise type can optionally customise the behaviour of every `co_await` expression that appears in the body of the coroutine.

promise 类型能定义出现在协程体内的每一个 `co_await` 表达式的行为。

By simply defining a method named `await_transform()` on the promise type, the compiler will then transform every `co_await <expr>` appearing in the body of the coroutine into `co_await promise.await_transform(<expr>)`.

简单的通过在 promise 类上定义一个 `await_transform()` 方法，编译器就会将协程体内的每一个 `co_await <expr>` 转换成 `co_await promise.await_transform(<expr>)` 。 

This has a number of important and powerful uses:

它具有很多强大且重要的用途：

**It lets you enable awaiting types that would not normally be awaitable.**

**它会使你能等待通常不能被等待的类型**。

For example, a promise type for coroutines with a `std::optional<T>` return-type might provide an `await_transform()` overload that takes a `std::optional<U>` and that returns an awaitable type that either returns a value of type `U` or suspends the coroutine if the awaited value contains `nullopt`.

例如，协程的 promise 类有一个 `std::optional<T>` 返回类型，该类型提供一接收 `std::optional<U>` 参数然后返回一个 awaitable 类型的 `await_transform()` 的重载，该 awaitable 对象要么返回类型 U 的值，要么当 等待的值包含 `nullopt` 时挂起协程。

```c++
template<typename T>
class optional_promise
{
  ...

  template<typename U>
  auto await_transform(std::optional<U>& value)
  {
    class awaiter
    {
      std::optional<U>& value;
    public:
      explicit awaiter(std::optional<U>& x) noexcept : value(x) {}
      bool await_ready() noexcept { return value.has_value(); }
      void await_suspend(std::experimental::coroutine_handle<>) noexcept {}
      U& await_resume() noexcept { return *value; }
    };
    return awaiter{ value };
  }
};
```

**It lets you disallow awaiting on certain types by declaring `await_transform` overloads as deleted.**

**通过声明 `await_transform` 重载为已删除，使你不允许等待某种类型**

For example, a promise type for `std::generator<T>` return-type might declare a deleted `await_transform()` template member function that accepts any type. This basically disables use of `co_await` within the coroutine.

例如，`std::generator<T>` 的 promise 类型的返回类型可能声明接收任意类型`await_transform` 模板成员函数被删除。这基本上禁用了 `co_await` 在协程内的使用。

```c++
template<typename T>
class generator_promise
{
  ...

  // Disable any use of co_await within this type of coroutine.
  template<typename U>
  std::experimental::suspend_never await_transform(U&&) = delete;

};
```

**It lets you adapt and change the behaviour of normally awaitable values**

**它使你能够适配和更改正常等待对象的行为**

For example, you could define a type of coroutine that ensured that the coroutine always resumed from every `co_await` expression on an associated executor by wrapping the awaitable in a `resume_on()` operator (see `cppcoro::resume_on()`).

例如，你可以定义一种协程类型，该协程类型通过将 awaitable 包装在 `resume_on()` 操作符中，确保协程始终从每个 `co_await` 表达式的关联执行者上恢复（参见 `cppcoro::resume_on()`）。

```c++
template<typename T, typename Executor>
class executor_task_promise
{
  Executor executor;

public:

  template<typename Awaitable>
  auto await_transform(Awaitable&& awaitable)
  {
    using cppcoro::resume_on;
    return resume_on(this->executor, std::forward<Awaitable>(awaitable));
  }
};
```

As a final word on `await_transform()`, it’s important to note that if the promise type defines *any* `await_transform()` members then this triggers the compiler to transform *all* `co_await` expressions to call `promise.await_transform()`. This means that if you want to customise the behaviour of `co_await` for just some types that you also need to provide a fallback overload of `await_transform()` that just forwards through the argument.

作为关于 `await_transform()` 的最后一句话，需要非常注意的是，如果promise 类型定义了任何 `await_transform` 成员，那么这将触发编译器将所有 `co_await` 表达式转化为 `promise.await_transform()` 调用。这意味着，如果你只想为某些类型自定义 `co_await` 的行为，你还需要提供仅通过参数转发的 `await_transform()` 重载。

### Customising the behaviour of `co_yield`

The final thing you can customise through the promise type is the behaviour of the `co_yield` keyword.

你可以通过 promise 类型自定义行为的最后一件事是 `co_yield` 关键字。

If the `co_yield` keyword appears in a coroutine then the compiler translates the expression `co_yield <expr>` into the expression `co_await promise.yield_value(<expr>)`. The promise type can therefore customise the behaviour of the `co_yield` keyword by defining one or more `yield_value()` methods on the promise object.

如果 `co_yield` 关键字出现在协程内，编译器会将 `co_yield <expr>` 表达式 转换为 `co_await promise.yield_value(<expr>)` 表达式。因此，promise 类型可以通过在 promise 对象上定义一个或多个 `yield_value()` 方法来定义 `co_yield` 关键字的行为。

Note that, unlike `await_transform`, there is no default behaviour of `co_yield` if the promise type does not define the `yield_value()` method. So while a promise type needs to explicitly opt-out of allowing `co_await` by declaring a deleted `await_transform()`, a promise type needs to opt-in to supporting `co_yield`.

注意，与 `await_transform` 不同，如果 promise 类型没有定义 `yield_value()` 方法，则 `co_yield` 没有默认行为。因此，虽然 promise 类型需要通过声明被删除的 `await_transform` 来显示地选择不允许 `co_await` ，但是 promise 类型需要选择支持 `co_yield`.

The typical example of a promise type with a `yield_value()` method is that of a `generator<T>` type:

带有 `yield_value()` 方法的 promise 类型典型用法是 `generator<T>`:

```c++
template<typename T>
class generator_promise
{
    T* valuePtr;
public:
    ...
        
        std::Experimental::suspend_always yield_value(T& value) noexcept
    {
        // Stash the address of the yield value and then return an awaitable
        // that will cause the coroutine to suspend at the co_yield express.
        // Execution will then return from the call to coroutine_handle<>::resume()
        // inside either generator<T>::begin() or generator<T>::iterator::operator++().
        valuePtr = std::addressof(value);
        return {};
    }
};
```

# Summary

In this post I’ve covered the individual transformations that the compiler applies to a function when compiling it as a coroutine.

在这篇文章中，我介绍了编译器将函数编译为协程时，编译器应用于该函数的各个转换。

Hopefully this post will help you to understand how you can customise the behaviour of different types of coroutines through defining different your own promise type. There are a lot of moving parts in the coroutine mechanics and so there are lots of different ways that you can customise their behaviour.

希望这篇文章能帮助你理解如何通过定义不同的 promise 类型来定制不同类型协程的行为。协程机制中有很多活动部件，因此有很多不同的方法可以定制它们的行为。

However, there is still one more important transformation that the compiler performs which I have not yet covered - the transformation of the coroutine body into a state-machine. However, this post is already too long so I will defer explaining this to the next post. Stay tuned!

但是，我还没有介绍另外一个编译器执行的非常重要的转换--将协程体转化为状态机。但是，这篇文章已经太长了，所以我将推迟到下一篇文章来解释。敬请期待。 

# C++ Coroutines: Understanding Symmetric Transfer

The Coroutines TS provided a wonderful way to write asynchronous code as if you were writing synchronous code. You just need to sprinkle `co_await` at appropriate points and the compiler takes care of suspending the coroutine, preserving state across suspend-points and resuming execution of the coroutine later when the operation completes.

Coroutines TS 提供了让你像写同步代码一样写异步代码的方法。你只需要在适当的位置写上 `co_await` ，编译器将负责挂起协程，保存挂起点的状态，并在操作完成后恢复协程的执行。

However, the Coroutines TS, as it was originally specified, had a nasty limitation that could easily lead to stack-overflow if you weren’t careful. And if you wanted to avoid this stack-overflow then you had to introduce extra synchronisation overhead to safely guard against this in your `task<T>` type.

但是，在最初的 Coroutines TS 中有一个令人讨厌的限制，你的不小心会很容易导致栈溢出。而且，如果你想避免这样的栈溢出，则必须在你的 `taks<T>` 类型中引入额外的同步开销来保证安全。

Thankfully, a tweak was made to the design of coroutines in 2018 to add a capability called “symmetric transfer” which allows you to suspend one coroutine and resume another coroutine without consuming any additional stack-space. The addition of this capability lifted a key limitation of the Coroutines TS and allows for much simpler and more efficient implementation of async coroutine types without sacrificing any of the safety aspects needed to guard against stack-overflow.

值得庆幸的是，2018 年对协程的设计进行了调整，增加了一个称为 “对称传输” 的功能，它允许你在不消耗任何额外栈空间的情况下暂停一个协程并恢复另外一个协程。此功能的添加解除了 Coroutines TS 的一个关键限制，允许更简单、有效地实现异步协程类型，而不牺牲任何防止栈溢出所需的安全措施。

In this post I will attempt to explain the stack-overflow problem and how the addition of this key “symmetric transfer” capability lets us solve this problem.

在这篇文章中，我将试图解释栈溢出问题，以及添加这种关键的 “对称传输” 功能是如何使我们解决这个问题的。

## First some background on how a task coroutine works

Consider the following coroutines:

考虑一下协程：

```c++
task foo() {
    co_return;
}

task bar() {
    co_await foo();
}
```

Assume we have a simple `task` type that lazily executes the body when another coroutine awaits it. This particular `task` type does not support returning a value.

假设我们有一个简单的 `task` 类型，当另外一个协程等待它时，它会懒执行自己的主体。该特定 `task` 类型不支持返回值。

Let’s unpack what’s happening here when `bar()` evaluates `co_await foo()`.

- The `bar()` coroutine calls the `foo()` function. Note that from the caller’s perspective a coroutine is just an ordinary function.
- The invocation of `foo()` performs a few steps:
  - Allocates storage for a coroutine frame (typically on the heap)
  - Copies parameters into the coroutine frame (in this case there are no parameters so this is a no-op).
  - Constructs the promise object in the coroutine frame
  - Calls `promise.get_return_object()` to get the return-value for `foo()`. This produces the `task` object that will be returned, initialising it with a `std::coroutine_handle` that refers to the coroutine frame that was just created.
  - Suspends execution of the coroutine at the initial-suspend point (ie. the open curly brace)
  - Returns the `task` object back to `bar()`.
- Next the `bar()` coroutine evaluates the `co_await` expression on the `task` returned from `foo()` .
  - The `bar()` coroutine suspends and then calls the `await_suspend()` method on the returned task, passing it the `std::coroutine_handle` that refers to `bar()`’s coroutine frame.
  - The `await_suspend()` method then stores `bar()`’s `std::coroutine_handle` in `foo()`’s promise object and then resumes the `foo()` coroutine by calling `.resume()` on `foo()`’s `std::coroutine_handle`.
- The `foo()` coroutine executes and runs to completion synchronously.
- The `foo()` coroutine suspends at the final-suspend point (ie. the closing curly brace) and then resumes the coroutine identified by the `std::coroutine_handle` that was stored in its promise object before it was started. ie. `bar()`’s coroutine.
- The `bar()` coroutine resumes and continues executing and eventually reaches the end of the statement containing the `co_await` expression at which point it calls the destructor of the temporary `task` object returned from `foo()`.
- The `task` destructor then calls the `.destroy()` method on `foo()`’s coroutine handle which then destroys the coroutine frame along with the promise object and copies of any arguments.

让我们拆解一下 `bar()` 中对 `co_await foo()` 求值时，发生了那些事情：

- 协程 `bar()` 调用了函数 `foo()`。注意，在调用者的角度来看，协程就是一个普通的函数。
- 调用 `foo()` 执行了这么几个步骤：
  - 为协程帧分配存储（通常在堆上）
  - 复制参数到协程帧（本例中没有参数，所以不需要该操作）
  - 在协程帧中构建 promise 对象
  - 调用 `promise.get_return_object()` 获取 `foo()` 的return-value。这会产生将要返回的 `task` 对象，并使用一个 `std::coroutine_handle` 对其进行初始化，该 `std::coroutine_handle` 对象引用刚被创建的协程帧。
  - 将协程在 initial-suspend 点挂起（即开始的花括号）
  - 返回 `task` 对象给 `bar()`。
- 接着，协程 `bar()` 的 `co_await` 表达式对 `foo()` 返回的 `task` 求值。
  - 协程 `bar()` 挂起，然后调用返回的 `task` 上的 `await_suspend()` 方法，并传入关联 `bar()` 协程帧的 `std::coroutine_handle` 。
  - `await_suspend()` 方法在 `foo()` 的 promise 对象中存储 `bar()` 的 `std::coroutine_handle` ，然后通过调用 `foo()` 的 `std::coroutine_handle` 上的 `.resume()` 方法恢复协程 `foo()`。
- 协程 `foo()` 同步的执行到结束。
- 协程 `foo()` 在 final-suspend 点（即结束的花括号）挂起，然后恢复之前存储在它的 promise 对象中的 `std::coroutine_handle` 所标识的协程，也就是协程 `bar()` 。
- 协程 `bar()` 恢复并继续执行，并最终到达包含 `co_await` 表达式的语句的末尾，然后它调用从 `foo()` 返回的临时 `task` 对象的析构函数。
- `task` 的析构函数调用 `foo()` 的协程句柄的 `.destroy()` 方法，然后销毁协程帧以及 promise 对象和所有参数的副本。

Ok, so that seems like a lot of steps for a simple call.

好吧，这对一个简单的调用来说似乎有太多的步骤了。

To help understand this in a bit more depth, let’s look at how a naive implementation of this `task` class would look when implemented using the the Coroutines TS design (which didn’t support symmetric transfer).

为了有助于更深入的了解这一点，让我们看看使用 Coroutines TS 设计（不支持对称传输）实现时，这个 `task` 类的有缺陷的实现是什么样子的。

## Outline of a `task` implementation

The outline of the class looks something like this:

该类大概类似这样：

```c++
class task{
public:
    class promise_type {/*see below*/};
    
    task(task&& t) noexcept
        :coro_(std::exchange(t.coro, {}))
    {}
    
    ~task() {
        if(coro_)
            coro_.destroy();
    }
    
    class awaiter { /*see below */};
    awaiter operator co_await()&& noexcept;
private:
    explicit task(std::coroutine_handle<promise_type> h) noexcept
        : coro_(h)
        {}
    std::coroutine_handle<promise_type> coro_;
};
```

A `task` has exclusive ownership of the `std::coroutine_handle` that corresponds to the coroutine frame created during the invocation of the coroutine. The `task` object is an RAII object that ensures that `.destroy()` is called on the `std::coroutine_handle` when the `task` object goes out of scope.

`task` 拥有 `std::coroutine_handle` 的独占权，该 `std::coroutine_handle` 对应调用协程时创建的协程帧。`task` 对象是一个 RAII 对象，他确保当 `task` 对象超出作用域时，会调用 `std::coroutine_handle` 上的 `.destroy()` 方法。

So now let’s expand on the `promise_type`.

现在，让我们扩展 `promise_type`。

## Implementing `task::promise_type`

From the [previous post](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type) we know that the `promise_type` member defines the type of the **Promise** object that is created within the coroutine frame and that controls the behaviour of the coroutine.

从[上一篇文章](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type)中我们知道，`promise_type` 成员定义了在协程帧中创建的 **Promise** 对象的类型，该 `Promise` 对象控制协程的行为。

First, we need to implement the `get_return_object()` to construct the `task` object to return when the coroutine is invoked. This method just needs to initialise the task with the `std::coroutine_handle` of the newly create coroutine frame.

首先，我们需要实现 `get_return_object()` 来构造调用协程时要返回的 `task` 对象。这个方法只需要用新创建的协程帧的 `std::coroutine_handle` 初始化 `task` 对象。

We can use the `std::coroutine_handle::from_promise()` method to manufacture one of these handles from the promise object.

我们可以使用 `std::coroutine_handle::from_promise()` 方法从 promise 对象中制造这些句柄之一。

```c++
class task::promise_type {
public:
    task get_return_object() noexcept {
        return task{std::coroutine_handle<promise_type>::from_promise(*this)};
    }
};
```

Next, we want the coroutine to initially suspend at the open curly brace so that we can later resume the coroutine from this point when the returned `task` is awaited.

接着，我们希望协程最初在开始的大括号处挂起，以便稍后返回的 `task` 变成等待时，我们可以从这里恢复协程。

There are several benefits of starting the coroutine lazily:

1. It means that we can attach the continuation’s `std::coroutine_handle` before starting execution of the coroutine. This means we don’t need to use thread-synchronisation to arbitrate the race between attaching the continuation later and the coroutine running to completion.
2. It means that the `task` destructor can unconditionally destroy the coroutine frame - we don’t need to worry about whether the coroutine is potentially executing on another thread since the coroutine will not start executing until we await it, and while it is executing the calling coroutine is suspended and so won’t attempt to call the task destructor until the coroutine finishes executing. This gives the compiler a much better chance at inlining the allocation of the coroutine frame into the frame of the caller. See [P0981R0](https://wg21.link/P0981R0) to read more about the Heap Allocation eLision Optimisation (HALO).
3. It also improves the exception-safety of your coroutine code. If you don’t immediately `co_await` the returned `task` and do something else that can throw an exception that causes the stack to unwind and the `task` destructor to run then we can safely destroy the coroutine since we know it hasn’t started yet. We aren’t left with the difficult choice between detaching, potentially leaving dangling references, blocking in the destructor, terminating or undefined-behaviour. This is something that I cover in a bit more detail in my [CppCon 2019 talk on Structured Concurrency](https://www.youtube.com/watch?v=1Wy5sq3s2rg).

延迟启动协程有这么几个好处：

1. 这意味着我们可以在执行协程之前附加持续的 `std::coroutine_handle`。这意味着我们不需要使用线程同步来仲裁稍后附加持续和运行到完成的协程间的竞争。
2. 这意味着 `task` 的析构函数可以无条件的销毁协程帧--我们不必担心协程是否有可能在另外一个线程上运行，因为协程在我们等待它之前不会开始执行，当它执行时，调用的协程将被挂起，因此在协程完成执行前不会尝试调用 `task` 的歇够函数。这使编译器有更好的机会将协程帧内联到调用者的帧中。有关堆分配省略优化（HALO）更多的信息，请参阅 [P0981R0](https://wg21.link/P0981R0)。
3. 它还能提供协程代码的异常安全性。如果你不立即 `co_await` 返回的 `task` 对象，并且执行了其他可能引发异常的操作，从而导致栈展开并执行了 `task` 的析构函数，那么我们可以安全的销毁协程，因为我们知道它还没有启动。在分离、潜在第留下挂起的引用、在析构函数中阻塞、终止或未定义的行为之间，我们没有面临艰难的选择。这是我在我的 [CppCon 2019 talk on Structured Concurrency](https://www.youtube.com/watch?v=1Wy5sq3s2rg) 中更详细讨论的内容。

To have the coroutine initially suspend at the open curly brace we define an `initial_suspend()` method that returns the builtin `suspend_always` type.

为了使协程在开始大括号处初始挂起，我们定义了 `initial_suspend()` 方法，该方法返回内置的 `suspend_always` 类型。

```c++
std::suspend_always initial_suspend() noexcept{
    return {};
}
```

Next, we need to define the `return_void()` method, called when you execute `co_return;` or when execution runs off the end of the coroutine. This method doesn’t actually need to do anything, it just needs to exist so that the compiler knows that `co_return;` is valid within this coroutine type.

接下来，我们需要定义 `return_void()` 方法，这个方法在执行 `co_return;` 时调用，或者运行到协程的末尾时调用。这个方法实际上不做任何事情，它只是需要存在，以便编译器知道 `co_return;` 在这个协程类型中是有效的。

```c++
void reutrn_void() noexcept {}
```

We also need to add an `unhandled_exception()` method that is called if an exception escapes the body of the coroutine. For our purposes we can just treat the task coroutine bodies as `noexcept` and call `std::terminate()` if this happens.

我们还需要添加一个 `unhandled_exception()` 方法，该方法在异常从协程体扩散出来时调用。出于我们的目的，我们可以仅将 task 协程体视为 `noexcept`  ，并在发生异常时调用 `std::terminate()` 。

```c++
void unhandled_exception() noexcept {
    std::terminate();
}
```

Finally, when the coroutine execution reaches the closing curly brace, we want the coroutine to suspend at the final-suspend point and then resume its continuation. ie. the coroutine that is awaiting the completion of this coroutine.

最后，当协程执行到结束的大括号时，我们希望协程在 final-suspend 点挂起，然后恢复它的执行，即等待这个协程完成的协程。

To support this, we need a data-member in the promise to hold the `std::coroutine_handle` of the continuation. We also need to define the `final_suspend()` method that returns an awaitable object that will resume the continuation after the current coroutine has suspended at the final-suspend point.

为了做到这一点，我们需要将协程的 `std::coroutine_handle`  保存在 `promise` 中的数据成员中。我们还需要定义 `final_suspend()` 方法，该方法返回一个可等待对象，该对象将在当前协程在 final-suspend 点挂起后继续执行后续的操作。

It’s important to delay resuming the continuation until after the current coroutine has suspended because the continuation may go on to immediately call the `task` destructor which will call `.destroy()` on the coroutine frame. The `.destroy()` method is only valid to call on a suspended coroutine and so it would be undefined-behaviour to resume the continuation before the current coroutine has suspended.

将恢复后续操作延迟到当前协程挂起后是很重要的，因为后续操作有可能会立即调用 `task`  的析构函数，这会导致在协程帧上调用 `.destroy()` 方法。`.destroy()` 方法只有在挂起的协程上调用才是有效的，因此在当前挂起协程之前恢复后续执行将产生未定义行为。

The compiler inserts code to evaluate the statement `co_await promise.final_suspend();` at the closing curly brace.

编译器在结束大括号处插入代码计算 `co_await promise.final_suepend();` 语句的值。

It’s important to note that the coroutine is not yet in a suspended state when the `final_suspend()` method is invoked. We need to wait until the `await_suspend()` method on the returned awaitable is called before the coroutine is suspended.

需要注意的是，当 `final_suspend()` 方法被调用时，协程还没有出于挂起状态。我们需要等到调用返回的 awaitable 上的 `await_suspend()` 方法后，才能挂起协程。

```c++
struct final_awaiter {
    bool await_ready() noexcept {
        return false;
    }
    
    void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
        // The coroutine is now suspended at the final-suspedn point.
        // Lookup its continuation in the promise and resume it.
        h.promise().continuation.resume();
    }
    
    void await_resume() noexcept {}
    
    final_awaiter final_suspend() noexcept {
        return {};
    }
    
    std::coroutine_handle<> continuation;
};
```

Ok, so that’s the complete `promise_type`. The final piece we need to implement is the `task::operator co_await()`.

好了，这就是完整的 promise_type。我们需要实现的最后一部分是 `task::operator co_await()`。

## Implementing `task::operator co_await()`

You may remember from the [Understanding operator co_await() post](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await) that when evaluating a `co_await` expression, the compiler will generate a call to `operator co_await()`, if one is defined, and then the object returned must have the `await_ready()`, `await_suspend()` and `await_resume()` methods defined.

你可能记得在 [理解 operator co_await() ](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await) 的文章中，当计算一个 `co_await()` 表达式时，编译器将生成一个对 `operator co_await()` 调用，如果定义了这个调用，那么返回的对象必须具有 `await_ready()`， `await_suspend()` 和 `await_resume()` 方法。

When a coroutine awaits a `task` we want the awaiting coroutine to always suspend and then, once it has suspended, store the awaiting coroutine’s handle in the promise of the coroutine we are about to resume and then call `.resume()` on the `task`’s `std::coroutine_handle` to start executing the task.

当一个协程等待一个 `task` 时，我们希望等待协程总是挂起，然后，一旦它挂起，将等待协程的句柄存储在我们将要恢复的协程的 `promise`  中，然后在 `task` 的 `std::coroutine_handle` 上调用 `.resume()` 方法，以开始执行该任务。

Thus the relatively straight forward code:

因此，相对简单的代码是：

```c++
class task::awaiter {
public:
    bool await_ready() noexcept {
        return false;
    }
    
    void await_suspend(std::coroutine_handle<> continuation) noexcept {
        // Store the continuation in the task's so that the final_suspend()
        // knows to resume this coroutine when the task complete
        coro_.promise().continuation = continuation;
        coro_.resume();
    }
    
    void await_resume() noexcept {}
private:
    explicit awaiter(std::coroutine_handle<task::promise_type> h) noexcept
        :coro_(h)
        {}
    std::coroutine_handle<task::promise_type> coro_;
};
task::awaiter task::operator co_await() && noexcept {
    return awaiter{coro_};
}
```

And thus completes the code necessary for a functional `task` type.

从而完成功能性 `task` 类型所需的代码。

You can see the complete set of code in Compiler Explorer here: https://godbolt.org/z/-Kw6Nf

你可以在此查看编译器资源管理器中的完整代码：https://godbolt.org/z/-Kw6Nf

## The stack-overflow problem

The limitation of this implementation arises, however, when you start writing loops within your coroutines and you `co_await` tasks that can potentially complete synchronously within the body of that loop.

然而，这种实现的局限在于，当你开始在协程中编写循环代码时，你的 `co_await` 任务可能会在循环体内同步的完成。

For example:

```c++
task completes_synchronously() {
    co_return;
}
task loop synchronously(int count) {
    for(int i = 0; i < count; ++i)
        co_await completes_synchronously();
}
```

With the naive `task` implementation described above, the `loop_synchronously()` function will (probably) work fine when `count` is 10, 1000, or even 100’000. But there will be a value that you can pass that will eventually cause this coroutine to start crashing.

对于上面描述的简单 `task` 的实现，当 `count` 为 10、1000 甚至是 100’000  时，`loop_synchronously()` 函数（可能）正常工作。但是你传递的值有可能导致这个协程开始崩溃。

For example, see: https://godbolt.org/z/gy5Q8q which crashes when `count` is 1’000’000.

例如，在 https://godbolt.org/z/gy5Q8q 这里你可以看到，当 `count` 是 1’000’000 时发生了崩溃。

The reason that this is crashing is because of stack-overflow.

这是因为栈溢出导致的崩溃。

To understand why this code is causing a stack-overflow we need to take a look at what is happening when this code is executing. In particular, what is happening to the stack-frames.

为了理解为什么这段代码会导致栈溢出，我们需要了解在执行这段代码时发生了什么。特别是，栈帧发生了什么。

When the `loop_synchronously()` coroutine first starts executing it will be because some other coroutine `co_await`ed the `task` returned. This will in turn suspend the awaiting coroutine and call `task::awaiter::await_suspend()` which will call `resume()` on the task’s `std::coroutine_handle`.

当 `loop_synchronously()` 协程第一次执行时，它将因为 `co_await` 其他协程而等待返回的 `task` 。这将反过来挂起等待的协程并调用 `task::awaiter::awaiter_suspend()` ，后者将调用 task 的 `std::coroutine_handle` 上的 `resume()` 方法。

Thus the stack will look something like this when `loop_synchronously()` starts:

因此，当 `loop_synchronously()` 启动时，栈看起来像下面一样：

```
           Stack                                                   Heap
+------------------------------+  <-- top of stack   +--------------------------+
| loop_synchronously$resume    | active coroutine -> | loop_synchronously frame |
+------------------------------+                     | +----------------------+ |
| coroutine_handle::resume     |                     | | task::promise        | |
+------------------------------+                     | | - continuation --.   | |
| task::awaiter::await_suspend |                     | +------------------|---+ |
+------------------------------+                     | ...                |     |
| awaiting_coroutine$resume    |                     +--------------------|-----+
+------------------------------+                                          V
|  ....                        |                     +--------------------------+
+------------------------------+                     | awaiting_coroutine frame |
                                                     |                          |
                                                     +--------------------------+
```

> Note: When a coroutine function is compiled the compiler typically splits it into two parts:
>
> 1. *the “ramp function” which deals with the construction of the coroutine frame, parameter copying, promise construction and producing the return-value, and*
> 2. the “coroutine body” which contains the user-authored logic from the body of the coroutine.
>
> I use the `$resume` suffix to refer to the “coroutine body” part of the coroutine.
>
> A later blog post will go into more detail about this split.

> 注意：当一个协程函数被编译时，编译器通常会把它分成两部分：
>
> 1. “ramp function” 处理协程帧的构造、参数复制、promise 构造和产生 return-value
> 2. “coroutine body” 包含协程体中用户编写的逻辑
>
> 我是用 `$resume` 后缀来代指协程的 `coroutine body` 部分。
>
> 稍后的博客文章会更详细的介绍这种分裂。

Then when `loop_synchronously()` awaits the `task` returned from `completes_synchronously()` the current coroutine is suspended and calls `task::awaiter::await_suspend()`. The `await_suspend()` method then calls `.resume()` on the coroutine handle corresponding to the `completes_synchronously()` coroutine.

然后，当 `loop_synchronously()` 等待 `completes_synchronously()` 返回的 `task`  ，当前协程被挂起并调用 `task::awaiter::await_suspend()`。然后，`await_suspend()` 方法在于 `completes_synchronously()` 协程相对应的协程句柄上调用 `.resume()` 。

This resumes the `completes_synchronously()` coroutine which then runs to completion synchronously and suspends at the final-suspend point. It then calls `task::promise::final_awaiter::await_suspend()` which calls `.resume()` on the coroutine handle corresponding to `loop_synchronously()`.

这将恢复 `completes_synchronously()` 协程，然后同步运行到完成并在 final_suspend 点挂起。然后，它调用 `task::promise::final_awaiter::await_suspend()` ，在对应于 `loop_synchronously()` 的协程句柄上调用 `.resume()` 。

The net result of all of this is that if we look at the state of the program just after the `loop_synchronously()` coroutine is resumed and just before the temporary `task` returned by `completes_synchronously()` is destroyed at the semicolon then the stack/heap should look something like this:

所有这一切的最终结果是，如果我们在 `loop_synchronously()` 协程恢复后，而在 `completes_synchronously()` 返回的临时 `task` 销毁之前，查看程序的状态，则堆栈应该是这样的：

```
           Stack                                                   Heap
+-------------------------------+ <-- top of stack
| loop_synchronously$resume     | active coroutine -.
+-------------------------------+                   |
| coroutine_handle::resume      |            .------'
+-------------------------------+            |
| final_awaiter::await_suspend  |            |
+-------------------------------+            |  +--------------------------+ <-.
| completes_synchronously$resume|            |  | completes_synchronously  |   |
+-------------------------------+            |  | frame                    |   |
| coroutine_handle::resume      |            |  +--------------------------+   |
+-------------------------------+            '---.                             |
| task::awaiter::await_suspend  |                V                             |
+-------------------------------+ <-- prev top  +--------------------------+   |
| loop_synchronously$resume     |     of stack  | loop_synchronously frame |   |
+-------------------------------+               | +----------------------+ |   |
| coroutine_handle::resume      |               | | task::promise        | |   |
+-------------------------------+               | | - continuation --.   | |   |
| task::awaiter::await_suspend  |               | +------------------|---+ |   |
+-------------------------------+               | - task temporary --|---------'
| awaiting_coroutine$resume     |               +--------------------|-----+
+-------------------------------+                                    V
|  ....                         |               +--------------------------+
+-------------------------------+               | awaiting_coroutine frame |
                                                |                          |
                                                +--------------------------+
```

Then the next thing this will do is call the `task` destructor which will destroy the `completes_synchronously()` frame. It will then increment the `count` variable and go around the loop again, creating a new `completes_synchronously()` frame and resuming it.

然后接下来要做的事情就是调用 `task` 的析构函数，销毁 `completes_synchronously()` 帧。然后，它将增加 `count` 变量并再次执行循环，创建一个新的 `completes_synchronously()` 帧并恢复它。

In effect, what is happening here is that `loop_synchronously()` and `completes_synchronously()` end up recursively calling each other. Each time this happens we end up consuming a bit more stack-space, until eventually, after enough iterations, we overflow the stack and end up in undefined-behaviour land, typically resulting in your program promptly crashing.

实际上，这里发生的是 `loop_synchronously()` 和 `completes_synchronously()`  互相调用对方直至递归结束。每次发生这种情况时，我们都会占用更多的栈空间，直到最终，经过足够多的迭代，我们溢出栈并最终导致未定义的行为，通常会导致程序迅速崩溃。

Writing loops in coroutines built this way makes it very easy to write functions that perform unbounded recursion without looking like they are doing any recursion.

在以这种方式构建的协程中编写循环，可以非常容易地编写执行无限递归的函数，而看起来却像没有执行递归一样。

So, what would the solution look like under the original Coroutines TS design?

那么，在原始的 Coroutines TS 设计下的解决方案会是什么样的呢。

## The Coroutines TS solution

Ok, so what can we do about this to avoid this kind of unbounded recursion?

那么，我们该怎么做才能避免这种无限制的递归呢。

With the above implementation we are using the variant of `await_suspend()` that returns `void`. In the Coroutines TS there is also a version of `await_suspend()` that returns `bool` - if it returns `true` then the coroutine is suspended and execution returns to the caller of `resume()`, otherwise if it returns `false` then the coroutine is immediately resumed, but this time without consuming any additional stack-space.

对于上面的实现，我们使用了 `await_suspend()` 的变体，该变体返回 `void` 。在 Coroutines TS 中，还有一个 `await_suspend()` 的版本，它返回 `bool` -- 如果它返回 `true` ，则协程被挂起，执行流返回到 `resume()`  的调用者，否则，如果它返回 `false` ，则立即恢复协程，但是这次不占用任何额外的栈空间。

So, to avoid the unbounded mutual recursion what we want to do is make use of the `bool`-returning version of `await_suspend()` to resume the current coroutine by returning `false` from the `task::awaiter::await_suspend()` method if the task completes synchronously instead of resuming the coroutine recursively using `std::coroutine_handle::resume()`.

因此，如果任务同步完成，为了避免无限的相互递归，我们要做的是利用返回 `bool` 的 `await_suspend()` 版本来恢复当前协程，方法是从 `task::awaiter::await_suspend()` 方法返回 `false` ，而不是使用 `std::coroutine_handle::resume()` 递归地恢复协程。

To implement a general solution for this there are two parts.

1. Inside the `task::awaiter::await_suspend()` method you can start executing the coroutine by calling `.resume()`. Then when the call to `.resume()` returns, check whether the coroutine has run to completion or not. If it has run to completion then we can return `false`, which indicates the awaiting coroutine should immediately resume, or we can return `true`, indicating that execution should return to the caller of `std::coroutine_handle::resume()`.
2. Inside `task::promise_type::final_awaiter::await_suspend()`, which is run when the coroutine runs to completion, we need to check whether the awaiting coroutine has (or will) return `true` from `task::awaiter::await_suspend()` and if so then resume it by calling `.resume()`. Otherwise, we need to avoid resuming the coroutine and notify `task::awaiter::await_suspend()` that it needs to return `false`.

要实现这个问题的通用解决方案，有两个问题：

1. 在 `task::awaiter::await_suspend()` 方法内部，你可以通过调用`.resume()` 方法开始执行协程。然后，当 `.resume()` 的调用返回时，检查协程是否执行完毕。如果已经执行完毕，则可以返回 `false` ，这表示等待的协程应立即恢复，或者可以返回 `true` ，表示执行应该返回给 `std::coroutine_handle::resume()` 的调用者。
2. 当协程运行到完成时，在运行的 `task::promise_type::final_awaiter::await_suspend()` 内部，我们需要检查等待协程是否已经（或将）从 `task::awaiter::await_suspend()` 返回 `true` ，如果是，则通过 `.resume()` 恢复它。否则，我们需要避免恢复协程，并通知 `task::awaiter::await_suspend()` 它需要返回 `false` 。

There is an added complication, however, in that it’s possible for a coroutine to start executing on the current thread then suspend and later resume and run to completion on a different thread before the call to `.resume()` returns. Thus, we need to be able to resolve the potential race between part 1 and part 2 above happening concurrently.

但是，这增加了以个复杂性，在对 `.resume()` 的调用返回之前，协程可能会在当前线程上开始执行，然后挂起，稍后在另外一个线程恢复并运行到完成。因此，我们需要能够解决上述第一部分和第二部分同时发生的潜在竞争。

We will need to use a `std::atomic` value to decide the winner of the race here.

我们将使用 `std::atomic`  值确定竞争的胜者。

Now for the code. We can make the following modifications:

现在，我们可以对代码做如下修改：

```c++
class task::promise_type {
    ...
        
    std::coroutine_handle<> continuation;
    std::atomic<bool> ready = false;
};

bool task::awaiter::await_suspend(
    std::coroutine_handle<> continuation) noexcept {
    promise_type& promise = coro_.promise();
    promise.continuation = continuation;
    coro_.resume();
    
    return !promise.ready.exchange(true, std::memory_order_acq_rel);
}

void task::promise_type::final_awaitre::await_suspend(
std::coroutine_handle<promise_type> h) noexcept {
    promise_type& promise = h.promise();
    if(promise.ready.exchange(true, std::memory_order_acq_rel)) {
        // The coroutine did not complete  synchronously, resume it here.
        promise.continuation.resume();
    }
}
```

See the updated example on Compiler Explorer: https://godbolt.org/z/7fm8Za Note how it no longer crashes when executing the `count == 1'000'000` case.

请参阅编译器资源管理器上的更新示例:  https://godbolt.org/z/7fm8Za 。注意，当 `count == 1'000'000`  例子时，它不在崩溃。

This turns out to be the approach that the `cppcoro::task<T>` [implementation](https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/task.hpp) took to avoid the unbounded recursion problem (and still does for some platforms) and it has worked reasonably well.

这就是 `cppcoro::task<T>` [实现](https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/task.hpp) 为了避免无限递归的问题而采用的方法（在某些平台上仍然是这样），而且效果不错。

Woohoo! Problem solved, right? Ship it! Right…?

哇哦! 问题解决了，对吧? 运走它! 对吧... ？

## The problems

While the above solution does solve the recursion problem it has a couple of drawbacks.

虽然上述解决方案确实解决了递归问题，但它有两个缺点。

Firstly, it introduces the need for `std::atomic` operations which can be quite costly. There is an atomic exchange on the caller when suspending the awaiting coroutine, and another atomic exchange on the callee when it runs to completion. If your application only ever executes on a single thread then you are paying the cost of the atomic operations for synchronising threads even though it’s never needed.

首先，它引入了对 `std::atomic` 操作的需求，这可能会非常昂贵。挂起等待协程时，在调用方上进行原子交还，而在被调用方运行完成时，在被调用方上进行原子交还。如果你的程序仅在单个线程上执行，那么即使你永远不需要它，也要为同步线程付出原子操作的开销。

Secondly, it introduces additional branches. One in the caller, which needs to decide whether to suspend or immediately resume the coroutine, and one in the callee, which needs to decide whether to resume the continuation or suspend.

其次，它引入了其他分支。一个在调用发中，它需要决定是暂停还是立即恢复协程，另一个在被调用方中，它需要决定继续还是暂停。

Note that the cost of this extra branch, and possibly even the atomic operations, would often be dwarfed by the cost of the business logic present in the coroutine. However, coroutines have been advertised as a zero cost abstraction and there have even been people using coroutines to suspend execution of a function to avoid waiting for an L1-cache-miss (see Gor’s great [CppCon talk on nanocoroutines](https://www.youtube.com/watch?v=j9tlJAqMV7U) for more details on this).

注意，这个额外分支的成本，甚至可能是原子操作的开销，与协程中的业务逻辑成本相比，往往相形见绌。然后，协程已经被宣传为一种零成本的抽象，甚至有人使用协程来暂停函数的执行，以避免等待 L1-cache-miss （一级缓存未命中）（有关这么方面的详细信息，请参阅 Gor 在 CppCon 关于 [nanocoroutines](https://www.youtube.com/watch?v=j9tlJAqMV7U) 的演讲）。

Thirdly, and probably most importantly, it introduces some non-determinism in the execution-context that the awaiting coroutine resumes on.

第三，也可能是最重要的是，它在协程恢复的执行上下文中引入了一些非确定性。

Let’s say I have the following code:

假设我有以下的代码：

```c++
cppcoro::static_thread_pool tp;

task foo()
{
    std::cout << "foo1" << std::this_thread::get_id() << "\n";
    // Suspend coroutine and reschedule onto thread-pool thread.
    co_await tp.schedule();
    std::cout << "foo2" << std::this_thread::get_id() << "\n";
}

task bar()
{
    std::cout << "bar1" << std::this_thread::get_id() << "\n";
    co_await foo();
    std::cout << "bar2" << std::this_thread::get_id() << "\n";
}
```

With the original implementation we were guaranteed that the code that runs after `co_await foo()` would run inline on the same thread that `foo()` completed on.

使用最初的实现，可以保证 `co_await foo()` 之后运行的代码可以在 `foo()` 完成的统一线程上运行。

For example, one possible output would have been:

例如，一种可能的输出是：

```
bar1 1234
foo1 1234
foo2 3456
bar2 3456
```

However, with the changes to use atomics, it’s possible the completion of `foo()` may race with the suspension of `bar()` and this can, in some cases, mean that the code after `co_await foo()` might run on the original thread that `bar()` started executing on.

但是，由于原子操作的引入， `foo()` 的完成和 `bar()` 的挂起可能产生竞争。在某些情况下，这意味着 `co_await foo()` 之后的代码可能会在 `bar()` 开始执行的原始线程上运行。

For example, the following output might now also be possible:

例如，现在也有可能出现以下输出：

```
bar1 1234
foo1 1234
foo2 3456
bar2 1234
```

For many use-cases this behaviour may not make a difference. However, for algorithms whose purpose is to transition execution context this can be problematic.

对于许多用例来说，这种行为可能没有什么不同。然而，对于目的是转换执行上下文的算法来说，这可能是个问题。

For example, the `via()` algorithm awaits some Awaitable and then produces it on the specified scheduler’s execution context. A simplified version of this algorithm is shown below.

例如，`via()` 算法等待某个 Awaitable ，然后在指定的调度程序的执行上下文中生成它。这个算法的简化版本如下所示。

```c++
template<typename Awaitable, typename Scheduler>
task<await_result_t<Awaitable>> via(Awaitable a, Scheduler s)
{
    auto result = co_await std::move(a);
    co_await s.schedule();
    co_return result;
}

task<T> get_value();
void consume(const T&);

task<void> consumer(static_thread_pool::scheduler s)
{
    T result = co_await via(get_value(), s);
    consume(result);
}
```

With the original version the call to `consume()` is always guaranteed to be executed on the thread-pool, `s`. However, with the revised version that uses atomics it’s possible that `consume()` might either be executed on a thread associated with the scheduler, `s`, or on whatever thread the `consumer()` coroutine started execution on.

对于原始版本，`consume()` 的调用始终保证在线程池中执行。但是，对于使用原子操作的修订版本， `consume()` 可能要么在与调度程序相关的线程上执行，要么在 `consume()` 协程开始执行的任何线程上执行。

So how do we solve the stack-overflow problem without the overhead of the atomic operations, extra branches and the non-deterministic resumption context?

那么，如何在不引入原子操作、额外分支和不确定性恢复上下文的情况下解决堆栈溢出的问题呢？

## Enter “symmetric transfer”!（对称传输）

The paper [P0913R0](https://wg21.link/P0913R0) “Add symmetric coroutine control transfer” by Gor Nishanov (2018) proposed a solution to this problem by providing a facility which allows one coroutine to suspend and then resume another coroutine symmetrically without consuming any additional stack-space.

Gor Nishanov （2018）的论文 [P0913R0](https://wg21.link/P0913R0) “添加对称协程控制传输” 提出了一种解决此问题的方法，即提供一种允许一个协程暂停然后对称地恢复另一个协程而不消耗任何额外栈空间的设施。

This paper proposed two key changes:

- Allow returning a `std::coroutine_handle<T>` from `await_suspend()` as a way of indicating that execution should be symmetrically transferred to the coroutine identified by the returned handle.
- Add a `std::experimental::noop_coroutine()` function that returns a special `std::coroutine_handle` that can be returned from `await_suspend()` to suspend the current coroutine and return from the call to `.resume()` instead of transferring execution to another coroutine.

论文提出了两项关键的改变：

- 允许从 `await_suspend() ` 返回 `std::coroutine_handle<T>` ，以指示执行流应该对称地转移到由返回的句柄标识的协程中。
- 添加一个 `std::experimental::noop_coroutine()` 函数，该函数返回一个特殊的 `std::coroutine_handle` ，可以从 `await_suspend()` 返回该特殊函数，从而挂起当前协程并从对 `.resume()` 的调用返回，而不是将执行转移到另一个协程。

So what do we mean by “symmetric transfer”?

那么，我们所说的“对称转移”是什么意思呢？

When you resume a coroutine by calling `.resume()` on it’s `std::coroutine_handle` the caller of `.resume()` remains active on the stack while the resumed coroutine executes. When this coroutine next suspends and the call to `await_suspend()` for that suspend-point returns either `void` (indicating unconditional suspend) or `true` (indicating conditional suspend) then call to `.resume()` will return.

当你通过在 `std::coroutine_handle` 上调用 `.resume()` 来恢复协程时，当被恢复的协程执行时，`.resume()`  的调用方在栈上保持活动状态。当次协程下一次挂起

并且对该挂起点的 `await_suspend()` 调用返回 `void` （表示无条件官气）或 `true` （表示有条件挂起），则将返回对 `.resume()` 的调用。

This can be thought of as an “asymmetric transfer” of execution to the coroutine and behaves just like an ordinary function call. The caller of `.resume()` can be any function (which may or may not be a coroutine). When that coroutine suspends and returns either `true` or `void` from `await_suspend()` then execution will return from the call to `.resume()` and

这可以看做是对协程执行的“非对称传输”，其行为和普通函数调用类似。`.resume()` 的调用者可以是任何函数（可以是也可以不是协程）。当协程挂起并从 `await_suspend()` 返回 `true` 和 `void` 时，执行将从对 `.resume()` 的调用返回。

Every time we resume a coroutine by calling `.resume()` we create a new stack-frame for the execution of that coroutine.

我们每次通过调用 `.resume()` 来恢复协程时，我们都会为协程的执行创建一个新的栈帧。

However, with “symmetric transfer” we are simply suspending one coroutine and resuming another coroutine. There is no implicit caller/callee relationship between the two coroutines - when a coroutine suspends it can transfer execution to any suspended coroutine (including itself) and does not necessarily have to transfer execution back to the previous coroutine when it next suspends or completes.

但是，通过“对称转移”，我们只是暂停一个协程并恢复另外一个协程。两个协程之间没有隐式的调用方/被调用方关系--当协程暂停时，它可以将执行转移到任何已暂停的协程（包括自身），并且不必再下一次暂停或完成时将执行转移回先前的协程。

Let’s look at what the compiler lowers a `co_await` expression to when the awaiter makes use of symmetric-transfer:

让我们来看一下，当 awaiter 使用对称传输时，编译器将 `co_await` 表达式降低到什么程度。

```c++
{
    decltype(auto) value = <exper>;
    decltype(auto) awaitable = 
        get_awaitable(promise, static_cast<decltype(value)&&>(value));
    decltype(auto) awaiter = 
        get_awaiter(static_cast<decltype(awaitable)&&>(awaitable));
    if(!awaiter.await_ready())
    {
        using handle_t = std::coroutine_handle<P>;
        
        //<suspend-coroutine>
        auto h = awaiter.await_suspend(handle_t::from_promise(p));
        h.resume();
        //<return-to-caller-or-resume>
        
        //<resume-point>
    }
    
    return awaiter.await_resume();
}
```

Let’s zoom in on the key part that differs from other `co_await` forms:

让我们放大与其他 `co_await` 形式不同的关键部分：

```c++
auto h = awaiter.await_suspend(handle_t::from_promise(p));
h.resume();
//<return-to-caller-or-resumer>
```

Once the coroutine state-machine is lowered (a topic for another post), the `<return-to-caller-or-resumer>` part basically becomes a `return;` statement which causes the call to `.resume()` that last resumed the coroutine to return to its caller.

一旦协程状态机降低（另一篇文章的主体），`<return-to-caller-or-resume>` 部分基本上就变成了 `return;` 语句，它使最后一次恢复协程的 `.resume()` 调用返回到它的调用者。

This means that we have the situation where we have a call to another function with the same signature, `std::coroutine_handle::resume()`, followed by a `return;` from the current function which is itself the body of a `std::coroutine_handle::resume()` call.

这意味着我们会遇到这样一种情况，即调用另一个具有相同签名的函数， `std::coroutine_handle::resume()` ，从当前本身就是一个 `std::coroutine_handle::resume()` 调用的函数返回。

Some compilers, when optimisations are enabled, are able to apply an optimisation that turns calls to other functions the tail-position (ie. just before returning) into tail-calls as long as some conditions are met.

启用优化功能后，某些编译器可以应用优化功能，只要瞒住某些条件，就可以将对其他函数的调用从尾部位置（即在返回之前）转换为尾部调用。

It just so happens that this kind of tail-call optimisation is exactly the kind of thing we want to be able to do to avoid the stack-overflow problem we were encountering before. But instead of being at the mercy of the optimiser as to whether or not the tail-call transformation is perfromed, we want to be able to guarantee that the tail-call transformation occurs, even when optimisations are not enabled.

碰巧的是，这种类型的尾部调用优化恰恰是我们希望能够做的事情，它可以避免我们以前遇到的栈溢出的问题。但是，我们希望能够保证即使在未启用优化的情况下，也会发生尾部调用转换，而不是由优化程序来决定是否执行尾部调用转换。

But first let’s dig into what we mean by tail-calls.

但首先，让我们深入了解尾部调用的含义。

### Tail-calls

A tail-call is one where the current stack-frame is popped before the call and the current function’s return address becomes the return-address for the callee. ie. the callee will return directly the the caller of this function.

尾部调用是在调用之前弹出当前的栈帧（被调用方），并把当前函数的返回地址变成被调用方的返回地址，即，被调用方直接将此函数返回给调用者。（调用者 --> 被调用方 --> 此函数）。

On X86/X64 architectures this generally means that the compiler will generate code that first pops the current stack-frame and then uses a `jmp` instruction to jump to the called function’s entry-point instead of using a `call` instruction and then popping the current stack-frame after the `call` returns.

在 X86/64 体系结构上，这通常意味着编译器将生成代码，该代码首先弹出当前栈帧，然后使用 `jmp` 指令跳转到被调用函数的入口点，而不是使用 `call` 指令，然后在 `call` 返回后弹出当前栈帧。

This optimisation is generally only possible to do in limited circumstances, however.

但是，这种优化通常只能在有限的情况下进行。

In particular, it requires that:

- the calling convention supports tail-calls and is the same for the caller and callee;
- the return-type is the same;
- there are no non-trivial destructors that need to be run after the call before returning to the caller; and
- the call is not inside a try/catch block.

特别是，它要求：

- 调用约定支持尾部调用，对调用者和被调用者是相同的；
- 返回类型相同；
- 在返回到调用方之前，不需要在调用之后运行非平凡的析构函数；
- 调用不在 try/catch 块中；

The shape of the symmetric-transfer form of `co_await` has actually been designed specifically to allow coroutines to satisfy all of these requirements. Let’s look at them individually.

实际上，对称转移形式的  `co_await` 的形状是经过专门设计的，以便协程可以满足这些要求。让我们单独看一下。

**Calling convention** When the compiler lowers a coroutine into machine code it actually splits the coroutine up into two parts: the ramp (which allocates and initialises the coroutine frame) and the body (which contains the state-machine for the user-authored coroutine body).

**Calling convention （调用约定）**。当编译器将协程编译为机器码时，它实际上将协程分为两个部分：ramp （斜坡）（分配和初始化协程帧）和主体（包含用户编写的协程主体的状态机）。

The function signature of the coroutine (and thus any user-specified calling-convention) affects only the ramp part, whereas the body part is under the control of the compiler and is never directly called by any user-code - only by the ramp function and by `std::coroutine_handle::resume()`.

协程的函数签名（以及任何用户指定的调用约定）只影响 ramp 部分，而主体部分由编译器控制，并且永远不会被任何用户代码直接调用--仅由 ramp 函数和 `std::coroutine_handle::resume()` 调用。

The calling-convention of the coroutine body part is not user-visible and is entirely up to the compiler and thus it can choose an appropriate calling convention that supports tail-calls and that is used by all coroutine bodies.

协程主体部分的调用约定不是用户可见的，完全取决于编译器，因此它可以选择支持尾部调用且所有协程主体都使用的合适的调用约定。

**Return type is the same** The return-type for both the source and target coroutine’s `.resume()` method is `void` so this requirement is trivially satisfied.

**Return type is the same（返回类型相同）**。 源协程和目标协程的 `.resume()` 方法的返回类型均是 `void` ，因此可以轻松满足此要求。

**No non-trivial destructors** When performing a tail-call we need to be able to free the current stack-frame before calling the target function and this requires the lifetime of all stack-allocated objects to have ended prior to the call.

**No non-trivial destructors（没有非平凡析构函数）**。当执行尾调用时，我们 需要能够在调用目标函数之前释放当前栈帧，这需要所有栈分配对象的生命周期在调用之前已经结束。

Normally, this would be problematic as soon as there are any objects with non-trivial destructors in-scope as the lifetime of those objects would not yet have ended and those objects would have been allocated on the stack.

通常，一旦作用域中存在任何具有非平凡析构函数的对象，就会出现问题，因为这些对象的生命周期还没有结束，而且这些对象已经在栈上分配了。

However, when a coroutine suspends it does so without exiting any scopes and the way it achieves this is by placing any objects whose lifetime spans a suspend-point in the coroutine frame rather than allocating them on the stack.

但是，当协程挂起时，它不会退出任何作用域，而实现这一点的方法是将任何生命周期跨越挂起点的对象放在协程帧中，而不是在栈中分配它们。

Local variables with lifetimes that do not span a suspend-point may be allocated on the stack, but the lifetime of these objects will have already ended and their destructors will have been called before the coroutine next suspends.

把生命周期不跨越挂起点的局部变量分配在栈上，在协程下次挂起之前，这些对象的生命周期已经结束并调用它们的析构函数。

Thus there should be no non-trivial destructors for stack-allocated objects that need to be run after the return of the tail-call.

因此，对于在尾调用返回之后需要运行的栈分配对象，不应该有非平凡析构函数。

**Call not inside a try/catch block** This one is a little tricker as within every coroutine there is an implicit try/catch block that encloses the user-authored body of the coroutine.

**Call not inside a try/catch block（不要在 try/catch 块中调用）**。这是一个技巧，因为每个协程中都有一个隐式的 try/catch 块，该块包围了用户编写的协程主体。 

From the specification, we see that the coroutine is defined as:

从规范中，我们可以看到协程的定义为：

```c++
{
    promise_type promise;
    co_await promise.initial_suspend();
    try { F;}
    catch(...) { promise.unhandled_exception(); }
final_suspend:
    co_await promise.final_suspend();
}
```

Where `F` is the user-authored part of the coroutine body.

其中 `F` 是用户编写的协程体的一部分。

Thus every user-authored `co_await` expression (other than initial/final_suspend) exists within the context of a try/catch block.

因此，每个用户编写的 `co_await` 表达式（除了 initial/final_suspend）都存在于 try/catch 块的上下文中。

However, implementations work around this by actually executing the call to `.resume()` *outside* of the context of the try-block.

但是，实现实际上是通过在 try 块的上下文之外执行对 `.resume()` 的调用来解决这个问题。

I hope to be able to go into this aspect in more detail in another blog post that goes into the details of the lowering of a coroutine to machine-code (this post is already long enough).

我希望能够在另一篇文章中更详细地讨论这方面的内容，其中涉及将协程编译成机器码的细节（这篇稳重已经够长了）。

> *Note, however, that the current wording in the C++ specification is not clear on requiring implementations to do this and it is only a non-normative note that hints that this is something that might be required. Hopefully we’ll be able to fix the specification in the future.*

> *然而，需要主要的是，在当前的 C++ 规范措辞中，并没有明确要求实现这样做，这只是一个非标准说明，暗示可能需要这样做。希望将来我们能够修复这个规范*

So we see that coroutines performing a symmetric-transfer generally satisfy all of the requirements for being able to perform a tail-call. The compiler guarantees that this will always be a tail-call, regardless of whether optimisations are enabled or not.

所以我们看到，执行对称传输的协程通常满足能够执行尾部调用的所有要求。编译器保证这将始终是一个尾部调用，而不管是否启用了优化。

This means that by using the `std::coroutine_handle`-returning flavour of `await_suspend()` we can suspend the current coroutine and transfer execution to another coroutine without consuming extra stack-space.

这意味着，通过使用 `std::coroutine_handle` 返回风格的 `await_suspend()` ，我们可以挂起当前的协程并将执行转移到另外一个协程，而无需占用额外的栈空间。

This allows us to write coroutines that mutually and recursively resume each other to an arbitrary depth without fear of overflowing the stack.

这使我们能编写协程，以相互递归的方式将彼此恢复到任意深度，而无需担心栈溢出。

This is exactly what we need to fix our `task` implementation

这正是我们解决 `task` 实现所需的。

## `task` revisited

So with the new “symmetric transfer” capability under our belt let’s go back and fix our `task` type implementation.

因此，借助新的“对称传输”功能，让我们回过头来解决我们的 `task` 类型设施。

To do this we need to make changes to the two `await_suspend()` methods in our implementation:

- First so that when we await the task that we perform a symmetric-transfer to resume the task’s coroutine.
- Second so that when the task’s coroutine completes that it performs a symmetric transfer to resume the awaiting coroutine.

为此，我们需要在实现中对两个 `await_suspend()` 方法进行更改：

- 首先，当我们等待 task 时，我们执行对称传输以恢复任务的协程。
- 其次，当任务的协程完成时，他执行对称传输以恢复等待的协程。

To address the await direction we need to change the `task::awaiter` method from this:

为了解决等待方向，我们需要将 `task::awaiter` 从这样：

```c++
void task::awaiter::await_suspend(
    std::coroutine_handle<> conitnuation) noexcept {
    // Store the continuation in the task's promise so that the final_suspend()
    // knows to resume this coroutine when the task completes.
    coro_.promise().continuation = continuation;
    
    // The we resume the task's coroutine, which is currently suspended
    // at the initial-suspend-point(ie, at the open curly brace).
    coro_.resume();
}
```

to this：

改成这样

```c++
std::coroutine_handle<> task::awaiter::await_suspend(
	std::coroutine_handle<> continuation) noexcept {
    // Store the continuation in the task's promise so that the final_suspend()
    // knows to resume this coroutine when the task complete.
    
    coro_.promise().continuation = continuation;
    
    // Then we tail-resume the task's coroutine, which is currently suspended
    // at the initial-suspend-point(ie, at the open curly brace), by returning
    // its handle from await_suspend().
    return coro_;
}
```

And to address the return-path we need to update the `task::promise_type::final_awaiter` method from this:

为了解决 return-path（返回路径），我们需要更新 `task::promise_type::final_awaiter` 方法从这样：

```c++
void task::promise_type::final_awaiter::await_suspend(
std::coroutine_handle<promise_type> h) noexcept {
    // The coroutine is now suspended at the final-suspend point.
    // Lookup its continuation in the promise and resume it.
    h.promise().continuation.resume();
}
```

to this:

到这样：

```c++
std::coroutine_handle<> task::promise_type::final_awaiter::await_suspend(
	std::coroutine_handle<promise_type> h) noexcept {
    // The coroutine is now suspended at the final-suspend point.
    // Lookup its continuation int the promise and resume it symmetrically.
    return h.promise().continuation;
}
```

And now we have a `task` implementation that doesn’t suffer from the stack-overflow problem that the `void`-returning `await_suspend` flavour had and that doesn’t have the non-deterministic resumption context problem of the `bool`-returning `await_suspend` flavour had.

现在我们有了一个 `task` 的实现，它没有 `void` 返回风格的 `await_suspend` 所具有的栈溢出问题，也没有 `bool` 返回风格的 `await_suspend` 所具有的不确定性恢复上下文问题。

### Visualising the stack

Let’s now go back and have a look at our original example:

现在，让我们回头看看原始示例：

```c++
task completes_synchronously() {
    co_return;
}

task loop_synchronously(int count) {
    for(int i = 0; i < count; ++i)
        co_await completes_synchronously();
}
```

When the `loop_synchronously()` coroutine first starts executing it will be because some other coroutine `co_await`ed the `task` returned. This will have been launched by symmetric transfer from some other coroutine, which would have been resumed by a call to `std::coroutine_handle::resume()`.

当 `loop_synchronously()` 协程第一次开始执行时，它将因为 `co_await` 其他协程  等待 `task` 返回。这将通过从其他协程的对称传输启动，通过调用 `std::coroutine_handle::resume()` 而恢复。

Thus the stack will look something like this when `loop_synchronously()` starts:

因此，当 `loop_synchronously()` 启动时，栈将如下所示：

```
           Stack                                                Heap
+---------------------------+  <-- top of stack   +--------------------------+
| loop_synchronously$resume | active coroutine -> | loop_synchronously frame |
+---------------------------+                     | +----------------------+ |
| coroutine_handle::resume  |                     | | task::promise        | |
+---------------------------+                     | | - continuation --.   | |
|     ...                   |                     | +------------------|---+ |
+---------------------------+                     | ...                |     |
                                                  +--------------------|-----+
                                                                       V
                                                  +--------------------------+
                                                  | awaiting_coroutine frame |
                                                  |                          |
                                                  +--------------------------+
```

Now, when it executes `co_await completes_synchronously()` it will perform a symmetric transfer to `completes_synchronously` coroutine.

现在，当它执行 `co_await completes_synchronously()` 时，它将执行对称传输到 `completes_synchronously()` 协程。

It does this by:

- calling the `task::operator co_await()` which then returns the `task::awaiter` object
- then suspends and calls `task::awaiter::await_suspend()` which then returns the `coroutine_handle` of the `completes_synchronously` coroutine.
- then performs a tail-call / jump to `completes_synchronously` coroutine. This pops the `loop_synchronously` frame before activing the `completes_synchronously` frame.

它通过以下方式做到这一点：

- 调用 `task::operator co_await()` ，然后返回 `task::awaiter` 对象。
- 然后挂起并调用`task::awaiter::await_suspend()` ，然后返回 `completes_synchronously` 协程的 `coroutine_handle`。
- 然后执行 tail-call/jump 到 `completes_synchronously` 协程。这将在激活 `completes_synchronously` 帧之前弹出 `loop_synchronously` 帧。

If we now look at the stack just after `completes_synchronously` is resumed it will now look like this:

现在，如果我们在 `completes_synchronously` 恢复后立即查看栈，它看起来是这样的：

```
              Stack                                          Heap
                                            .-> +--------------------------+ <-.
                                            |   | completes_synchronously  |   |
                                            |   | frame                    |   |
                                            |   | +----------------------+ |   |
                                            |   | | task::promise        | |   |
                                            |   | | - continuation --.   | |   |
                                            |   | +------------------|---+ |   |
                                            `-, +--------------------|-----+   |
                                              |                      V         |
+-------------------------------+ <-- top of  | +--------------------------+   |
| completes_synchronously$resume|     stack   | | loop_synchronously frame |   |
+-------------------------------+ active -----' | +----------------------+ |   |
| coroutine_handle::resume      | coroutine     | | task::promise        | |   |
+-------------------------------+               | | - continuation --.   | |   |
|     ...                       |               | +------------------|---+ |   |
+-------------------------------+               | task temporary     |     |   |
                                                | - coro_       -----|---------`
                                                +--------------------|-----+
                                                                     V
                                                +--------------------------+
                                                | awaiting_coroutine frame |
                                                |                          |
                                                +--------------------------+
```

Note that the number of stack-frames has not grown here.

注意，此处的栈帧并未增加。

After the `completes_synchronously` coroutine completes and execution reaches the closing curly brace it will evaluate `co_await promise.final_suspend()`.

在 `completes_synchronously` 协程完成并且执行到大括号后，它将计算 `co_await promise.final_suspend()`。

This will suspend the coroutine and call `final_awaiter::await_suspend()` which return the continuation’s `std::coroutine_handle` (ie. the handle that points to the `loop_synchronously` coroutine). This will then do a symmetric transfer/tail-call to resume the `loop_synchronously` coroutine.

这将挂起协程并调用 `final_awaiter::await_suspend()` ，该函数返回延续的 `std::coroutine_handle` （即指向 `loop_synchronously` 协程的句柄）。然后，这将执行对称传输/尾调用去恢复 `loop_synchronously` 协程。

If we look at the stack just after `loop_synchronously` is resumed then it will look something like this:

如果我们在 `loop_synchronously` 恢复之后立即查看栈，那么它看起来像这样：

```
           Stack                                                   Heap
                                                   +--------------------------+ <-.
                                                   | completes_synchronously  |   |
                                                   | frame                    |   |
                                                   | +----------------------+ |   |
                                                   | | task::promise        | |   |
                                                   | | - continuation --.   | |   |
                                                   | +------------------|---+ |   |
                                                   +--------------------|-----+   |
                                                                        V         |
+----------------------------+  <-- top of stack   +--------------------------+   |
| loop_synchronously$resume  | active coroutine -> | loop_synchronously frame |   |
+----------------------------+                     | +----------------------+ |   |
| coroutine_handle::resume() |                     | | task::promise        | |   |
+----------------------------+                     | | - continuation --.   | |   |
|     ...                    |                     | +------------------|---+ |   |
+----------------------------+                     | task temporary     |     |   |
                                                   | - coro_       -----|---------`
                                                   +--------------------|-----+
                                                                        V
                                                   +--------------------------+
                                                   | awaiting_coroutine frame |
                                                   |                          |
                                                   +--------------------------+
```

The first thing the `loop_synchronously` coroutine is going to do once resumed is to call the destructor of the temporary `task` that was returned from the call to `completes_synchronously` when execution reaches the semicolon. This will destroy the coroutine-frame, freeing its memory and leaving us with the following sitution:

一旦恢复，`loop_synchronously` 协程要做的第一件事就是在执行到达分号，将从调用 `compeltes_synchronously` 返回的临时 `task` 的析构函数。这将销毁协程帧，释放其内存，并使我们出于如下状态：

```
           Stack                                                   Heap
+---------------------------+  <-- top of stack   +--------------------------+
| loop_synchronously$resume | active coroutine -> | loop_synchronously frame |
+---------------------------+                     | +----------------------+ |
| coroutine_handle::resume  |                     | | task::promise        | |
+---------------------------+                     | | - continuation --.   | |
|     ...                   |                     | +------------------|---+ |
+---------------------------+                     | ...                |     |
                                                  +--------------------|-----+
                                                                       V
                                                  +--------------------------+
                                                  | awaiting_coroutine frame |
                                                  |                          |
                                                  +--------------------------+
```

We are now back to executing the `loop_synchronously` coroutine and we now have the same number of stack-frames and coroutine-frames as we started, and will do so each time we go around the loop.

现在我们又回到执行 `loop_synchronously` 协程，并且现在我们拥有与开始时相同数量的栈帧和协程帧，并且每次循环时都会这么做。

Thus we can perform as many iterations of the loop as we want and will only use a constant amount of storage space.

因此，我们可以根据需要执行任意多次循环，并且仅使用恒定数量的存储空间。

For a full example of the symmetric-transfer version of the `task` type see the following Compiler Explorer link: https://godbolt.org/z/9baieF.

有关 `task` 类型的对称传输版本的完整示例，请参阅一下编译器资源管理器链接： https://godbolt.org/z/9baieF。

## Symmetric Transfer as the Universal Form of await_suspend

Now that we see the power and importance of the symmetric-transfer form of the awaitable concept, I want to show you that this form is actually the universal form, which can theoretically replace the `void` and `bool`-returning forms of `await_suspend()`.

现在我们看到了对称转移形式的 awaitable 概念的威力和重要性，我想告诉你，这种形式实际上是一种普遍形式，理论上可以取代 `void` 和 `bool` 返回形式的 `await_suspend()`。

But first we need to look at the other piece that the [P0913R0](https://wg21.link/P0913R0) proposal added to the coroutines design: `std::noop_coroutine()`.

但是，首先我们需要看看 [P0913R0](https://wg21.link/P0913R0) 提案添加到协程设计中的另外一部分：`std::noop_coroutine()`

### Terminating the recursion

With the symmetric-transfer form of coroutines, every time the coroutine suspends it symmetrically resumes another coroutine. This is great as long as you have another coroutine to resume, but sometimes we don’t have another coroutine to execute and just need to suspend and let execution return to the caller of `std::coroutine_handle::resume()`.

利用协程的对称传输形式，每次协程被对称地挂起时，都会对称地恢复另外一个协程。只要你还有另外一个协程要继续执行，这都是可以的，但是有时候我们没有另外一个协程要执行，只需要挂起并让执行返回到 `std::coroutine_handle::resume()` 的调用方。

Both the `void`-returning and `bool`-returning flavours of `await_suspend()` allow a coroutine to suspend and return from `std::coroutine_handle::resume()`, so how do we do that with the symmetric-transfer flavour?

`void` 和 `bool` 返回形式的 `await_suspend()` 都允许协程挂起并从 `std::coroutine_handle::resume()` 返回，那么如何使用对称传输风格的 `await_suspend()` 实现这一点呢？

The answer is by using the special builtin `std::coroutine_handle`, called the “noop coroutine handle” which is produced by the function `std::noop_coroutine()`.

答案是使用特殊的内置的 `std::coroutine_handle`，被称为 “noop coroutine handle”，它由 `std::noop_coroutine()` 函数生成。

The “noop coroutine handle” is named as such because its `.resume()` implementation is such that it just immediately returns. i.e. resuming the coroutine is a no-op. Typically its implementation contains a single `ret` instruction.

之所以成为 “noop coroutine handle”，是因为它的 `.resume()` 实现使它只会立即返回，即，恢复协程时是一个无操作的（no-op）。它的实现通常只包含一条 `ret` 指令。

If the `await_suspend()` method returns the `std::noop_coroutine()` handle then instead of transferring execution to the next coroutine, it transfers execution back to the caller of `std::coroutine_handle::resume()`.

如果 `await_suspend()` 方法返回 `std::noop_coroutine()` 句柄，那么它将执行传递回 `std::coroutine_handle::resume()` 的调用方，而不是将执行传递到下一个 协程。

### Representing the other flavours of `await_suspend()`

With this information in-hand we can now show how to represent the other flavours of `await_suspend()` using the symmetric-transfer form.

有了这些信息，我们现在可以展示如何使用对称传输形式的 `await_suspend()`。

The `void`-returning form

`void` 返回形式：

```c++
void my_awaiter::await_suspend(std::coroutine_handle<> h) {
    this->coro = h;
    enqueue(this);
}
```

can also be written using both the `bool`-returning form:

也可以使用 `bool` 返回形式编写：

```c++
bool my_awaiter::await_suspend(std::coroutine_handle<> h) {
    this->coro = h;
    enqueue(this);
    return true;
}
```

and can be written using the symmetric-transfer form:

并且也能使用对称传输形式编写：

```c++
std::noop_coroutine_handle my_awaiter::await_suspend(
	std::coroutine_handle<> h) {
    this->coro = h;
    enqueue(this);
    return std::noop_coroutine();
}
```

The `bool`-returning form:

`bool` 返回形式：

```c++
bool my_awaiter::await_suspend(std::coroutine_handle<> h) {
    this->coro = h;
    if(try_start(this)) {
        // Operation will complete asynchronously.
        // Return true to transfer execution on caller of
        // coroutine_handle::resume().
        return true;
    }
    
    //Operation completed synchronously.
    // Return false to immediately resume the current coroutine.
    return false;
}
```

can also be written using the symmetric-transfer form:

也可以使用对称传输形式编写：

```c++
std::coroutine_handle<> my_awaiter::await_suspend(std::coroutine_handle<> h) {
    this->coro = h;
    if(try_start(this)) {
        // Operation will complete asynchronously.
        // Return std::noop_coroutine() to transfer exeution to caller of
        // coroutine_handle::resume().
        
        return std::noop_coroutine();
    }
 
    // Operation completed synchronously.
    // Return current coroutine's handle to immediately resume
    // the current coroutine.
    return h;
}
```

### Why have all three flavours?

So why do we still have the `void` and `bool`-returning flavours of `await_suspend()` when we have the symmetric-transfer flavour?

那么，为什么当我们已经有了对称传输风格的 `await_suspend()` 时，我们仍然还有 `void` 和 `bool` 返回风格的 `await_suspend()` 呢。

The reason is partly historical, partly pragmatic and partly performance.

究其原因，一部分是历史原因，一部分是实用，还有一部分是性能。

The `void`-returning version could be entirely replaced by returning the `std::noop_coroutine_handle` type from `await_suspend()` as this would be an equivalent signal to the compiler that the coroutine is unconditionally transfering execution to the caller of `std::coroutine_handle::resume()`.

 返回 `void` 的版本可以完全替换为从 `await_suspend()` 返回 `std::noop_coroutine_handle` 类型的版本，因为这相当于向编译器发出信号，表明协程正在无条件的将执行转移到 `std::coroutine_handle::resume()` 的调用方。

That it was kept was, IMO, partly because it was already in-use prior to the introduction of symmetric-transfer and partly because the `void`-form results in less-code/less-typing for the unconditional suspend case.

IMO 之所以保留它，部分是因为它在引入对称传输之前就已经在使用了，部分原因是因为 `void` 形式在无条件挂起时，拥有更少的代码。

The `bool`-returning version, however, can have a slight win in terms of optimisability in some cases compared to the symmetric-transfer form.

然而，在某些情况下，与对称传输形式相比，`bool` 返回版本在优化性方面可能稍有胜算。

Consider the case where we have a `bool`-returning `await_suspend()` method that is defined in another translation unit. In this case the compiler can generate code in the awaiting coroutine that will suspend the current coroutine and then conditionally resume it after the call to `await_suspend()` returns by just executing the next piece of code. It knows exactly the piece of code to execute next if `await_suspend()` returns `false`.

考虑这样一种情况，`bool` 返回形式的 `await_suspend()` 方法定义在另外一个翻译单元中。在这种情况下，编译器可以在等待的协程中生成代码，该协程将挂起当前协程，然后在调用 `await_suspend()` 返回后，仅执行下一段代码即可有条件的恢复它。如果 `await_suspend()` 返回 `false` ，它将确切知道接下来要执行的代码。

With the symmetric-transfer flavour we still need to represent the same outcomes; either return to the caller/resume or resume the current coroutine. Instead of returning `true` or `false` we need to return `std::noop_coroutine()` or the handle to the current coroutine. We can coerce both of these handles into a `std::coroutine_handle<void>` type and return it.

对于对称传输风格，我们仍然需要标识相同的结果；要么返回到调用者/恢复者，要么恢复当前的协程。代替返回`true` 或 `false` ，我们需要返回 `std::noop_coroutine()` 或当前协程的句柄。我们可以将这两个句柄强制转换为 `std::coroutine_handle<void>` 类型并返回它。

However, now, because the `await_suspend()` method is defined in another translation unit the compiler can’t see what coroutine the returned handle is referring to and so when it resumes the coroutine it now has to perform some more expensive indirect calls and possibly some branches to resume the coroutine, compared to a single branch for the `bool`-returning case.

但是，现在，由于 `await_suspend()` 方法是在另外一个翻译单元中定义的，编译器看不到返回的句柄所指的协程，因此当它恢复协程时，它现在必须执行一些更昂贵的间接调用，并可能需要一些分支来恢复协程，相比之下，`bool` 返回形式的情况只需要一个分支。

Now, it’s possible that we might be able to get equivalent performance out of the symmetric transfer version one day. For example, we could write our code in such a way that `await_suspend()` is defined inline but calls a `bool`-returning method that is defined out-of-line and then conditionally returns the appropriate handle.

现在，也许有一天我们可以从对称传输版本中获得同等的性能。例如，我们可以以如下方式编写代码：`await_suspend()` 是内联定义的，但是调用一个不是内联定义的 `bool` 返回形式的方法，然后有条件的返回适当的句柄。

For example:

例如

```c++
struc my_awaiter {
    bool await_ready();
    
    // Compilers should in-theory be able to optimise this to the same
    // as the bool-returning version, but currently don't do this optimisation.
    std::coroutine_handle<> await_suspend(std::coroutine_handle<> h) {
        if(try_start(h)) {
            return std::noop_coroutine();
        } else {
            return h;
        }
    }
    
    void await_resume();
private:
    // This method is defined out-of-line in a separate translation unit.
    bool try_start(std::coroutine_handle<> h);
}
```

However, current compilers (c. Clang 10) are not currently able to optimise this to as efficient code as the equivalent `bool`-returning version. Having said that, you’re probably not going to notice the difference unless you’re awaiting this in a really tight loop.

但是，当前的编译器（c. Clang 10）目前还不能将其优化为与等效的 `bool` 返回形式版本那样高效的代码。话虽如此，你可能不会注意到区别，除非你在一个非常紧密的循环中等待这个。

So, for now, the general rule is:

- If you need to unconditionally return to `.resume()` caller, use the `void`-returning flavour.
- If you need to conditionally return to `.resume()` caller or resume current coroutine use the `bool`-returning flavour.
- If you need to resume another coroutine use the symmetric-transfer flavour.

因此，目前，一般规则是：

- 如果需要无条件地返回给 `.resume()` 的调用者，请使用 `void` 返回风格的版本。
- 如果需要有条件地返回给 `.resume()` 的调用者或恢复当前的协程，请使用 `bool` 返回风格的版本。
- 如果需要恢复其他协程，请使用对称传输风格的版本。

# Rounding out

The new symmetric transfer capability added to coroutines for C++20 makes it much easier to write coroutines that recursively resume each other without fear of running into stack-overflow. This capability is key to making efficient and safe async coroutine types, such as the `task` one presented here.

在 C++20 的协程中添加了新的对称传输功能，这使得编写递归地相互恢复的协程变得更加容易，而不用担心会遇到栈溢出。这种能力是创建高效、安全的异步协程类型的关键，例如本文提到的 `task`。

This ended up being a much longer than expected post on symmetric transfer. If you made it this far, then thanks for sticking with it! I hope you found it useful.

这是一个比预期长的多的关于对称传输的文章。如果你做到了这一步，那么谢谢你坚持下去。希望你觉得有用。

In the next post, I’ll dive into understanding how the compiler transforms a coroutine function into a state-machine.

在下一篇文章中，我将深入介绍编译器如何将协程函数转换为状态机的。

# Thanks

Thanks to Eric Niebler and Corentin Jabot for providing feedback on drafts of this post.

感谢 Niebler 和 Corentin Jabot对本文草稿的反馈。