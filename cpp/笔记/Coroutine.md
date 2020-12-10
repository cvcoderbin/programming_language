# Coroutine Theory （协程理论）

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

**Call** 操作将会创建一个 activation frame，挂起( **suspends**)调用函数的执行，并将执行权转移到被调用函数的开头。

The **Return** operation passes the return-value to the caller, destroys the activation frame and then resumes execution of the caller just after the point at which it called the function.

**Return** 操作将返回值(**return-value**)传递给函数的调用者，销毁 activation frame并将执行权交还给调用函数，调用函数从调用位置后恢复( **resumes** )执行。

Let’s analyse these semantics a little more…

让我们再分析一下这些语义。。。

### Activation Frames（活动/活跃帧）

So what is this ‘activation frame’ thing?

那么什么是“Activation Frames” 呢？

You can think of the activation frame as the block of memory that holds the current state of a particular invocation of a function. This state includes the values of any parameters that were passed to it and the values of any local variables.

你可以将 activation frames 视为一个内存块，它维护了调用函数的当前状态。这些状态包括传递给它的所有的参数值和局部变量。

For “normal” functions, the activation frame also includes the return-address - the address of the instruction to transfer execution to upon returning from the function - and the address of the activation frame for the invocation of the calling function. You can think of these pieces of information together as describing the ‘continuation’ of the function-call. ie. they describe which invocation of which function should continue executing at which point when this function completes.

对于“普通”函数而言，activation frames 还包含返回地址(**return-address**):

- 从函数返回时，调用函数将要执行的下一条指令的地址
- 调用函数的 activation frame 地址

你可以认为这些信息一起描述了函数调用的 “continuation（继续性）”，即它们描述了当一个函数完成时，应该在哪个位置继续执行哪一个函数的调用。

With “normal” functions, all activation frames have strictly nested lifetimes. This strict nesting allows use of a highly efficient memory allocation data-structure for allocating and freeing the activation frames for each of the function calls. This data-structure is commonly referred to as “the stack”.

对于“普通”函数来说，所有的 activation frames 都拥有严格的嵌套生命周期。这种严格的嵌套，允许使用高效的内存分配数据结构来为每个函数调用分配和释放activation frames。这种数据结构通常称为“栈”。

When an activation frame is allocated on this stack data structure it is often called a “stack frame”.

当一个 activation frame 被分配在栈上时，它通常被称为 “stack frame”。

This stack data-structure is so common that most (all?) CPU architectures have a dedicated register for holding a pointer to the top of the stack (eg. in X64 it is the `rsp` register).

这种栈结构非常普遍，以至于大多数（或者全部？）CPU 体系架构都有一个专门的寄存器，用于保存指向栈顶部的指针（例如，在X64中，它是rsp寄存器）。

To allocate space for a new activation frame, you just increment this register by the frame-size. To free space for an activation frame, you just decrement this register by the frame-size.

当为一个新的 activation frame 申请空间时，你只需要将该寄存器按帧大小递增即可。要释放 activation frame 的空间时，只要将该寄存器按帧大小减小即可。

### The ‘Call’ Operation

When a function calls another function, the caller must first prepare itself for suspension.

当一个函数调用另外一个函数时，调用者(**Caller**) 必须先做好挂起自身的准备。

This ‘suspend’ step typically involves saving to memory any values that are currently held in CPU registers so that those values can later be restored if required when the function resumes execution. Depending on the calling convention of the function, the caller and callee may coordinate on who saves these register values, but you can still think of them as being performed as part of the **Call** operation.

“挂起（suspend）”步骤通常是将当前保存在 CPU 寄存器的值保存到内存中，以便在函数恢复执行时可以根据需要还原这些值。根据函数的调用约定，调用者（**Caller**）和被调用者（**Callee**）可以协调谁保存这些寄存器值，但是你仍可以将它们视为 **Call** 操作的一部分。

The caller also stores the values of any parameters passed to the called function into the new activation frame where they can be accessed by the function.

调用者还会将传递给被调用函数的任何参数值存储到新的 activation frame中，以便在被调用函数中访问。

Finally, the caller writes the address of the resumption-point of the caller to the new activation frame and transfers execution to the start of the called function.

最后，调用者将自己的恢复点（resumption-point ）地址写入新的 activation frame，并将执行流转移到被调用函数的开头。

In the X86/X64 architecture this final operation has its own instruction, the `call` instruction, that writes the address of the next instruction onto the stack, increments the stack register by the size of the address and then jumps to the address specified in the instruction’s operand.

在 X86/X64 架构中，这个最后的操作有自己的执行，即 **Call** 指令，它将下一条指令地址写入栈顶部，按地址大小增加栈寄存器，然后跳转到指令操作数中指定的地址。

### The ‘Return’ Operation

When a function returns via a `return`-statement, the function first stores the return value (if any) where the caller can access it. This could either be in the caller’s activation frame or the function’s activation frame (the distinction can get a bit blurry for parameters and return values that cross the boundary between two activation frames).

当函数通过返回语句(return-statement)返回时，该函数会先将返回值（如果有）存储在调用者可以访问的地方。返回值可能存储在调用者或者函数的 activation frame 中（这样来区分，会让跨越两个activation frames的参数和返回值变得有点模糊）。

Then the function destroys the activation frame by:

- Destroying any local variables in-scope at the return-point.
- Destroying any parameter objects
- Freeing memory used by the activation-frame

然后，函数通过以下步骤销毁activation frame：

- 销毁返回点范围内的所有局部变量。
- 销毁所有的参数对象
- 释放 activation frame 占用的内存

And finally, it resumes execution of the caller by:

- Restoring the activation frame of the caller by setting the stack register to point to the activation frame of the caller and restoring any registers that might have been clobbered by the function.
- Jumping to the resume-point of the caller that was stored during the ‘Call’ operation.

最后，它通过以下方式恢复调用者的执行：

- 将栈寄存器指针指向调用者的activation frame，并恢复任何可能被调用函数破坏的寄存器，来恢复调用者的activation frame。
- 跳转到 “**Call**” 操作期间，保存起来的调用者恢复点。

Note that as with the ‘Call’ operation, some calling conventions may split the repsonsibilities of the ‘Return’ operation across both the caller and callee function’s instructions.

注意，和 **Call** 操作一样，某些约定可能会在 **Caller** 和 **Callee** 之间划分 **Return** 操作的责任。

## Coroutines

Coroutines generalise the operations of a function by separating out some of the steps performed in the **Call** and **Return** operations into three extra operations: **Suspend**, **Resume** and **Destroy**.

协程泛化了函数的操作，将 **Call** 和 **Return** 操作中执行的一些步骤，又新定义了三个额外的操作：**Suspend**，**Resume** 和 **Destroy**。

The **Suspend** operation suspends execution of the coroutine at the current point within the function and transfers execution back to the caller or resumer without destroying the activation frame. Any objects in-scope at the point of suspension remain alive after the coroutine execution is suspended.

**Suspend** 操作可以在函数内部将协程挂起，并在不破坏 activation frame 的情况下将执行流交还给调用者或者resumer 。协程挂起后，挂起点内的所有对象仍然是可用的。

Note that, like the **Return** operation of a function, a coroutine can only be suspended from within the coroutine itself at well-defined suspend-points.

注意，就像函数的 **Return** 操作，协程只能在定义好的暂停点处从协程内部暂停。

The **Resume** operation resumes execution of a suspended coroutine at the point at which it was suspended. This reactivates the coroutine’s activation frame.

**Resume** 操作恢复执行一个在挂起点挂起的协程。这将重新激活协程的 activation frame。

The **Destroy** operation destroys the activation frame without resuming execution of the coroutine. Any objects that were in-scope at the suspend point will be destroyed. Memory used to store the activation frame is freed.

**Destroy** 操作销毁协程的 activation frame 并且不恢复协程的执行。挂起点范围内的所有对象都将被销毁。用于保存 activation frame 的内存也将被释放。

### Coroutine activation frames

Since coroutines can be suspended without destroying the activation frame, we can no longer guarantee that activation frame lifetimes will be strictly nested. This means that activation frames cannot in general be allocated using a stack data-structure and so may need to be stored on the heap instead.

由于协程可以在不破坏 activation frame 的情况下被挂起，我们将不再保证 activation frame 的生命周期被严格嵌套。这意味着 activation frames 通常不能像以前一样保存在栈中，因此可能需要将其存储在堆中。

There are some provisions in the C++ Coroutines TS to allow the memory for the coroutine frame to be allocated from the activation frame of the caller if the compiler can prove that the lifetime of the coroutine is indeed strictly nested within the lifetime of the caller. This can avoid heap allocations in many cases provided you have a sufficiently smart compiler.

如果编译器可以证明协程的生命周期确实严格嵌套在调用者的生命周期内，则 C++ Coroutine TS 中有一些规定允许从调用者的 activation frame 中为协程帧（ coroutine frame）分配内存。这样，许多情况下就可以避免从堆中分配内存了。

With coroutines there are some parts of the activation frame that need to be preserved across coroutine suspension and there are some parts that only need to be kept around while the coroutine is executing. For example, the lifetime of a variable with a scope that does not span any coroutine suspend-points can potentially be stored on the stack.

对于协程来说，activation frame 的某些部分是需要在被挂起时保存，而有些部分只需要在协程执行时保持。例如，作用于内生命周期不跨越任何协程挂起点的变量可以保存在栈中。

You can logically think of the activation frame of a coroutine as being comprised of two parts: the ‘coroutine frame’ and the ‘stack frame’.

你可以从逻辑上认为协程的 activation frame 包含 **coroutine frame** 和 **stack frame** 两部分。

The ‘coroutine frame’ holds part of the coroutine’s activation frame that persists while the coroutine is suspended and the ‘stack frame’ part only exists while the coroutine is executing and is freed when the coroutine suspends and transfers execution back to the caller/resumer.

coroutine frame 维护了一部分协程的 activation frame，该 coroutine frame 在协程被挂起时仍然存在，而 stack frame 仅在协程执行时存在，在协程被挂起并将执行流转移到 Caller 和 resumer 时释放。

### The ‘Suspend’ operation

The **Suspend** operation of a coroutine allows the coroutine to suspend execution in the middle of the function and transfer execution back to the caller or resumer of the coroutine.

协程的 **Suspend** 操作允许协程协程在函数中间暂停执行，并将执行权交还给 协程的 caller 和 resumer。

There are certain points within the body of a coroutine that are designated as suspend-points. In the C++ Coroutines TS, these suspend-points are identified by usages of the `co_await` or `co_yield` keywords.

协程内的某些点被称为挂起点。在 C++ Coroutine TS 中，这些挂起点是通过是`co_await` 或 `co_yield `关键字标识的。

When a coroutine hits one of these suspend-points it first prepares the coroutine for resumption by:

- Ensuring any values held in registers are written to the coroutine frame
- Writing a value to the coroutine frame that indicates which suspend-point the coroutine is being suspended at. This allows a subsequent **Resume** operation to know where to resume execution of the coroutine or so a subsequent **Destroy** to know what values were in-scope and need to be destroyed.

当协程到达这些挂起点时，它通过以下步骤为以后恢复协程做准备：

- 将寄存器中保存的值写入 coroutine frame。
- 向协程的 coroutine frame 写入一个值，该值指示协程的挂起点。这使后续的 **Resume** 操作知道该从哪里恢复执行协程，或者让后续的 **Destroy** 操作知道哪些在范围内的值需要被销毁。

Once the coroutine has been prepared for resumption, the coroutine is considered ‘suspended’.

一旦协程准备完毕，协程就被认为是“suspended”的了。

The coroutine then has the opportunity to execute some additional logic before execution is transferred back to the caller/resumer. This additional logic is given access to a handle to the coroutine-frame that can be used to later resume or destroy it.

在执行流转回交给 caller 或 resume 之前，协程有机会执行一些其他逻辑。这些额外逻辑被用来访问 coroutine frame 的句柄，该句柄可以在以后用来恢复或销毁协程。

This ability to execute logic after the coroutine enters the ‘suspended’ state allows the coroutine to be scheduled for resumption without the need for synchronisation that would otherwise be required if the coroutine was scheduled for resumption prior to entering the ‘suspended’ state due to the potential for suspension and resumption of the coroutine to race. I’ll go into this in more detail in future posts.

这种在协程进入 “Suspended” 状态后执行逻辑的能力允许将协程调度到恢复状态，而不需要同步，如果协程在进入 “Suspended” 状态之前被调度到执行恢复操作，则将需要同步，这是因为协程有可能在挂起和恢复操作中产生潜在的竞争。我将在以后的文章中更详细地讨论这个问题。

The coroutine can then choose to either immediately resume/continue execution of the coroutine or can choose to transfer execution back to the caller/resumer.

协程可以选择立即恢复/继续执行协程，也可以选择将执行流转移到 caller 和 resumer。

If execution is transferred to the caller/resumer the stack-frame part of the coroutine’s activation frame is freed and popped off the stack.

如果执行流被转交到 caller / resumer，则释放协程 activation frame 的 stack-frame 部分，并将其从栈中弹出。

### The ‘Resume’ operation

The **Resume** operation can be performed on a coroutine that is currently in the ‘suspended’ state.

可以在当前处于 “Suspended” 状态的协程上执行**Resume** 操作。

When a function wants to resume a coroutine it needs to effectively ‘call’ into the middle of a particular invocation of the function. The way the resumer identifies the particular invocation to resume is by calling the `void resume()` method on the coroutine-frame handle provided to the corresponding **Suspend** operation.

当一个函数想要恢复协程时，它需要“调用”协程的特定调用，也就是调用相应的 **Suspend** 操作的 coroutine-frame句柄提供的 `void resume()`方法。

Just like a normal function call, this call to `resume()` will allocate a new stack-frame and store the return-address of the caller in the stack-frame before transferring execution to the function.

就像普通函数调用一样，调用 resume() 将会分配一个新的 stack-frame，并在将执行流交到该函数之前(`resume()`)将调用者的返回地址储存到 stack-frame中。

However, instead of transferring execution to the start of the function it will transfer execution to the point in the function at which it was last suspended. It does this by loading the resume-point from the coroutine-frame and jumping to that point.

但是，它不是将执行权移交到函数的开始，而是将执行权转移到上次的挂起点。这是通过从coroutine-frame加载恢复点并跳到这一点实现的。

When the coroutine next suspends or runs to completion this call to `resume()` will return and resume execution of the calling function.

当协程下一次挂载或执行完毕时，这个对resume()的调用将返回并恢复对调用函数的执行。

### The ‘Destroy’ operation

The **Destroy** operation destroys the coroutine frame without resuming execution of the coroutine.

**Destroy** 操作销毁协程帧，并且不恢复协程的执行。

This operation can only be performed on a suspended coroutine.

这个操作只能在挂起的协程上执行。

The **Destroy** operation acts much like the **Resume** operation in that it re-activates the coroutine’s activation frame, including allocating a new stack-frame and storing the return-address of the caller of the **Destroy** operation.

**Destroy** 操作与 **Resume** 操作非常相似，它会重新激活协程的 activation frame，包括分配新的 stack-frame和存储 **Destroy** 操作调用者的返回地址。 

However, instead of transferring execution to the coroutine body at the last suspend-point it instead transfers execution to an alternative code-path that calls the destructors of all local variables in-scope at the suspend-point before then freeing the memory used by the coroutine frame.

然后，它不是将执行流移交给最后一次的挂起点，而是将执行权转交到另一个代码路径，该代码路径在挂起点作用域内调用所有局部变量的析构函数，然后释放协程帧使用的内存。

Similar to the **Resume** operation, the **Destroy** operation identifies the particular activation-frame to destroy by calling the `void destroy()` method on the coroutine-frame handle provided during the corresponding **Suspend** operation.

与 **Resume** 操作相似，**Destroy** 操作是通过调用处于 **Suspend** 的coroutine-frame句柄上提供的 `void destroy()` 函数销毁特定 activation-frame。

### The ‘Call’ operation of a coroutine

The **Call** operation of a coroutine is much the same as the call operation of a normal function. In fact, from the perspective of the caller there is no difference.

协程的 **Call** 操作和普通函数的 **Call** 操作基本相同。事实上，站在调用者的角度来说，两者完全相同。

However, rather than execution only returning to the caller when the function has run to completion, with a coroutine the call operation will instead resume execution of the caller when the coroutine reaches its first suspend-point.

但是，和函数仅在自己执行完毕时才将执行权交还给调用者不同，协程在到达第一个挂起点时，**Call**  操作将恢复调用者的执行。

When performing the **Call** operation on a coroutine, the caller allocates a new stack-frame, writes the parameters to the stack-frame, writes the return-address to the stack-frame and transfers execution to the coroutine. This is exactly the same as calling a normal function.

当在一个协程上执行 **Call** 操作时，调用者将分配一个新的 stack-frame，并将参数和调用者的返回地址写入 stack-frame，然后将执行流交给协程。这与调用一个普通函数完全相同。

The first thing the coroutine does is then allocate a coroutine-frame on the heap and copy/move the parameters from the stack-frame into the coroutine-frame so that the lifetime of the parameters extends beyond the first suspend-point.

协程要做的第一件事就是在堆中分配一个 coroutine-frame，然后将参数从stack-frame 复制/移动到 coroutine-frame，这样参数的生命周期就可以超出第一次的挂起点了。

### The ‘Return’ operation of a coroutine

The **Return** operation of a coroutine is a little different from that of a normal function.

协程的 **Return** 操作和普通函数的 **Return** 操作略有不同。

When a coroutine executes a `return`-statement (`co_return` according to the TS) operation it stores the return-value somewhere (exactly where this is stored can be customised by the coroutine) and then destructs any in-scope local variables (but not parameters).

当协程执行返回语句（TS规范中的 `co_return`）操作时，它会将返回值储存到某个地方（协程可以自定义这个值的存储位置），然后销毁作用域内的任何局部变量（不是参数）。

The coroutine then has the opportunity to execute some additional logic before transferring execution back to the caller/resumer.

在将执行流转移到 caller/resumer 之前，协程可以执行一些额外的逻辑。

This additional logic might perform some operation to publish the return value, or it might resume another coroutine that was waiting for the result. It’s completely customisable.

这些逻辑可以执行一些逻辑去发布返回值，或者恢复另外一个等待结果的协程。这是可自定义的。

The coroutine then performs either a **Suspend** operation (keeping the coroutine-frame alive) or a **Destroy** operation (destroying the coroutine-frame).

然后协程可以执行 **Suspend** 操作（保持 coroutine-frame 存活）或者执行 **Destroy** 操作（销毁coroutine-frame）。

Execution is then transferred back to the caller/resumer as per the **Suspend**/**Destroy** operation semantics, popping the stack-frame component of the activation-frame off the stack.

之后，按照 **Suspend** /**Destroy** 操作语义将执行流转交回 caller/resumer，再从栈中弹出 activation-frame 的 stack-frame。

It is important to note that the return-value passed to the **Return** operation is not the same as the return-value returned from a **Call** operation as the return operation may be executed long after the caller resumed from the initial **Call** operation.

需要注意的是，传递给 **Return** 操作的返回值和从 **Call** 操作返回的返回值不相同，因为 **Return** 操作可能在 caller 从初始 **Call** 操作恢复很久之后才执行。

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

然后，当调用`x(42)`时，它像普通函数一样，先创建 `x()` 的 stack frame。

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

然后，一旦协程 `x()`在堆上为协程分配了内存，并将参数值复制/移动到 coroutine frame 后，我们将得到类似于下一张图中的内容。注意，编译器通常会将coroutine frame的地址保存在单独记录栈顶指针的寄存器中（MSVC将保存在rbp寄存器中）。

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

当 `g()` 返回时，`g()` 的 activation frame将被销毁，然后恢复 `x()` 的 activation frame。假设我们将 `g()` 的返回值保存到一个存储在coroutine frame中的局部变量`b`  中。

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

如果 `x()` 执行到了挂起点，它将挂起并且不去销毁自己的 activation frame，然后将执行流交回 `f()`。

This results in the stack-frame part of `x()` being popped off the stack while leaving the coroutine-frame on the heap. When the coroutine suspends for the first time, a return-value is returned to the caller. This return value often holds a handle to the coroutine-frame that suspended that can be used to later resume it. When `x()` suspends it also stores the address of the resumption-point of `x()` in the coroutine frame (call it `RP` for resume-point).

这将导致 `x()` 的stack-frame 部分从栈中弹出，同时将 coroutine frame 保留在堆中。当协程第一次挂起时，一个返回值将被返回给调用者。这个返回值通常包含被挂起协程的 coroutine-frame 的一个句柄，它可以被用来在之后恢复协程的执行。当 `x()` 被挂起时，它也会将 `x()` 的恢复点地址存储在 coroutine frame中（RP表示恢复点）。

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

这将创建一个新的 stack-frame，它记录了调用者调用 `resume()` 时的地址 ，加载coroutine-frame 的地址到寄存器，找到存储在 coroutine-frame 中的恢复点，然后在此恢复执行 `x()`。

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

- 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
- 本文永久链接：https://github.com/xitu/gold-miner/blob/master/TODO1/understanding-operator-co-await.md
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

为了在需要的地方更具体的表述，我喜欢使用术语 **Normally Awaitable** 来描述promise类型中没有 `await_transform` 成员的协程~~上下文中支持 `co_await` 运算符的~~类型。我喜欢使用术语 **Contextually Awaitable** 来描述一种协程类型~~，它在某些类型的协程的上下文中仅支持 `co_await` 运算符~~，它的 promise 类型中存在 `await_transform` 方法。（我乐意接受其他更贴切的名字。。。）。

An **Awaiter** type is a type that implements the three special methods that are called as part of a `co_await` expression: `await_ready`, `await_suspend` and `await_resume`.

**Awaiter** 类型是一种实现了三个特殊方法的类型：`await_read`， `await_suspend` 和 `await_resume`，它们是 `co_await` 表达式的一部分。

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

因此，假设我们已经分装了将 `<expr>` 结果转换为 **Awaiter** 对象到上述函数中的逻辑，那么 `co_await <expr>` 的语义可以（大致）这样转换：

```c++
{
    auto&& value = <expr>;
    auto&& awaitable = get_awaitable(promise,static_cast<decltype(value)>(value));
    auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
    
    if(!awaiter.await_ready())
    {
        using handle_t = std::experimental::coroutine_handle<P>;
        using await_suspend_t = decltype(awaiter.await_suspend(handle_t::from_promise(p)));
        <suspend-coroutine>
         
        if constexpr (std::is_void_v<await_suspend_result_t>)
        {
            awaiter.await_suspend(handle_t::from_promise(p));
            <return-to-caller-or-resumer>
        }
        else
        {
            static_assert(
            	std::is_same_v<await_suspend_result_t,bool>,
                "await_suspend() must return 'void' or 'bool'.");
            if(awaiter.await_suspend(handle_t::from_promise(p)))
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

当 `await_suspend()` 的调用返回时，`await_suspend()` 的返回值为 `void` 的版本无条件地将执行转移回协程的调用者/恢复者，而返回值为 `bool` 的版本允许  awaiter 对象有条件地返回并立即恢复协程，而不是返回调用者/恢复者。

The `bool`-returning version of `await_suspend()` can be useful in cases where the awaiter might start an async operation that can sometimes complete synchronously. In the cases where it completes synchronously, the `await_suspend()` method can return `false` to indicate that the coroutine should be immediately resumed and continue execution.

`await_suspend()` 的 `bool` 返回版本在 awaiter 可能启动异步操作（有时可能同步完成）的情况下非常有用。在它同步完成的情况下，`await_suspend()` 方法可以返回 `false` 以指示应该立即恢复协程并继续执行。

At the `<suspend-coroutine>` point the compiler generates some code to save the current state of the coroutine and prepare it for resumption. This includes storing the location of the `<resume-point>` as well as spilling any values currently held in registers into the coroutine frame memory.

在 `<suspend-coroutine>` 处，编译器生成一些代码来保存协程的当前状态并准备恢复。这包括存储 `<resume-point>` 的断点位置，以及将当前保存在寄存器中的任何值溢出到协程帧内容中。

The current coroutine is considered suspended after the `<suspend-coroutine>` operation completes. The first point at which you can observe the suspended coroutine is inside the call to `await_suspend()`. Once the coroutine is suspended it is then able to be resumed or destroyed.

在 `<suspend_coroutine>` 操作完成后，当前的协程被认为是暂停的。你可以观察到暂停的协程的第一个断点是在 `await_suspend()` 的调用中。协程暂停后，就可以恢复或销毁。

It is the responsibility of the `await_suspend()` method to schedule the coroutine for resumption (or destruction) at some point in the future once the operation has completed. Note that returning `false` from `await_suspend()` counts as scheduling the coroutine for immediate resumption on the current thread.

当操作完成后，`await_suspend()` 方法负责在将来的某个时刻调度并将协程恢复（或销毁）。注意，从 `await_suspend()` 返回 `false` 算作调度协程，以便在当前线程上立即恢复。

The purpose of the `await_ready()` method is to allow you to avoid the cost of the `<suspend-coroutine>` operation in cases where it is known that the operation will complete synchronously without needing to suspend.

`await_ready()` 方法的目的，是允许你在已知操作同步完成而不需要挂起的情况下避免 `<suspend-coroutine>` 操作的成本。

At the `<return-to-caller-or-resumer>` point execution is transferred back to the caller or resumer, popping the local stack frame but keeping the coroutine frame alive.

在 `<return-to-caller-or-resumer>` 断点处执行转移回调用者或恢复者，弹出本地栈帧但是保持协程帧。

When (or if) the suspended coroutine is eventually resumed then the execution resumes at the `<resume-point>`. ie. immediately before the `await_resume()` method is called to obtain the result of the operation.

当（或者说如果）暂停的协程最终恢复时，执行将在 `<resume-point>` 断点处重新开始。即紧接在调用 `await_resume()` 方法获取操作结果之前。

The return-value of the `await_resume()` method call becomes the result of the `co_await` expression. The `await_resume()` method can also throw an exception in which case the exception propagates out of the `co_await` expression.

`await_resume()` 方法调用的返回值成为 `co_await` 表达式的结果。`await_resume()` 方法也可以抛出异常，在这种情况下异常从 `co_await` 表达式中抛出。

Note that if an exception propagates out of the `await_suspend()` call then the coroutine is automatically resumed and the exception propagates out of the `co_await` expression without calling `await_resume()`.

注意，如果异常从 `await_suspend()` 抛出，则协程会自动恢复，并且异常会从`co_await` 表达式抛出而不调用 `await_resume()`。

## Coroutine Handles

You may have noticed the use of the `coroutine_handle<P>` type that is passed to the `await_suspend()` call of a `co_await` expression.

你可能已经注意到 `coroutine-handle<P>` 类型的使用，该类型被传递给 `co_await` 表达式的 `await_suspend()` 调用。

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

`coroutine_handle<P>::from_promise(P& promise)` 函数允许从对协程的 promise 对象的引用重构协程句柄。注意，你必须确保类型 `P` 与用于协程帧的具体 promise 类型完全匹配；当具体的 promise 类型时 `Derived` 时，试图通过 `coroutine_handle<Base>` 类型构造协程句柄将会出现未定义行为的错误。

The `.address()` / `from_address()` functions allow converting a coroutine handle to/from a `void*` pointer. This is primarily intended to allow passing as a ‘context’ parameter into existing C-style APIs, so you might find it useful in implementing **Awaitable** types in some circumstances. However, in most cases I’ve found it necessary to pass additional information through to callbacks in this ‘context’ parameter so I generally end up storing the `coroutine_handle` in a struct and passing a pointer to the struct in the ‘context’ parameter rather than using the `.address()` return-value.

`.address()` / `from_address()` 函数允许协程句柄和 `void*` 指针相互转化。这主要是为了允许作为 “context” 参数传递到现有的 C 风格 API 中，因此你可能会发现在某些情况下实现 **Awaitable** 类型很有用。但是，在大多数情况下，我发现有必要将附加信息传递给这个“context” 参数中的回调，因此我通常最终将 `coroutine_handle` 存储在结构中并将指针传递给 “context” 参数中的结构而不是使用 `.address()` 返回值。

## Synchronisation-free async code

One of the powerful design-features of the `co_await` operator is the ability to execute code after the coroutine has been suspended but before execution is returned to the caller/resumer.

`co_await` 运算符的一个强大的设计功能是在协程挂起后和执行权返回给执行者/恢复者之前执行代码的能力。

This allows an Awaiter object to initiate an async operation after the coroutine is already suspended, passing the `coroutine_handle` of the suspended coroutine to the operation which it can safely resume when the operation completes (potentially on another thread) without any additional synchronisation required.

这允许 Awaiter 对象在协程已经被挂起后发起异步操作，将被挂起的协程的句柄 `coroutine_handle` 传递给运算符，当操作完成时（可能在另外一个线程上）它可以安全地恢复协程，而不需要额外的同步。

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

对于许多有栈协程框架，一个协程的挂起操作与另一个协程的恢复操作相结合，形成一个“context-switch（上下文切换）”操作。使用这种“context-switch”操作，通常在当前协程挂起后，而在将执行转移到另外一个协程之前，没有机会执行逻辑。

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

//A single call to produce a value
void producer()
{
    value = some_long_running_computation();
    //Publish the value by setting the event.
    event.set();
}

// Support multiple concurrent consumers
task<> consumer()
{
    // Wait until the event is signalled by call to event.set() in the producer() function.
    co_await event;
    
    // Now it's sage to consume 'value'
    // This is guaranteed to 'happen after' assignment to 'value'
    std::cout << value << std::endl;
}
```

Let’s first think about the possible states this event can be in: ‘not set’ and ‘set’.

让我们首先考虑一下这个事件可能存在的状态： “not set” 和 “set”。

When it’s in the ‘not set’ state there is a (possibly empty) list of waiting coroutines that are waiting for it to become ‘set’.

当它处于 “not set” 状态时，有一队（可能为空）协程正在等待它变为 “set” 状态。

When it’s in the ‘set’ state there won’t be any waiting coroutines as coroutines that `co_await` the event in this state can continue without suspending.

当它处于 “set” 状态时，不会有任何等待的协程，因为在状态下， `co_await` 的事件可以继续执行不用暂停。

This state can actually be represented in a single `std::atomic<void*>`.

- Reserve a special pointer value for the ‘set’ state. In this case we’ll use the `this` pointer of the event since we know that can’t be the same address as any of the list items.
- Otherwise the event is in the ‘not set’ state and the value is a pointer to the head of a singly linked-list of awaiting coroutine structures.

这个状态实际上可以用一个 `std::atomic<void*>` 来表示。

- 为 “set” 状态保留一个特殊的指针值。在这种情况下，我们将使用事件的 `this` 指针，因为我们知道该地址不会与任何列表项相同的地址。
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
    async_manual_reset_event& operator(async_manual_reset_event&&) = delete;
    
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
    //Special m_state value that indicates the event is in the 'set' state
    const void* const setState = &m_event;
    
    // Remember the handle of the awaiting coroutine.
    m_awaitingCoroutine = awaitingCoroutine;
    
    // Try to atomically push this awaiter onto the front of the list
    void* oldValue = m_evnet.m_state.load(std::memory_order_acquire);
    do
    {
        // Resume immediately if already in 'set' state
        if(oldValue == setState) return false;
        m_next = static_cast<awaiter*>(oldValue);
        
        // Finally, try to swap the old list head, inserting this awaiter as the new list head.
    }
    while(!m_event.m_state.compare_exchange_weak(
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
    if(oldValue != this)
    {
        // Wasn't already in 'set' state
        // Treat old value as head of a linked-list of waiters.
        // which we hava now acquired and need to resume
        auto* waiters = static_cast<awaiter*>(oldValue);
        while(waiters != nullptr)
        {
            // Read m_next befor resuming the coroutine as resuming
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