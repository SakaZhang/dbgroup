- > Coroutines定义: `协程是一个允许被挂起和可以被稍后恢复的函数的泛型抽象`。
## 普通函数

一个普通的函数可以被认为有两个操作：Call和Return（请注意，在这里将“抛出异常”广义地归为返回操作）。Call 操作创建一个活动帧，暂停调用函数的执行并将执行转移到被调用函数的开头。Return 操作将返回值传递给调用者，销毁活动帧，然后在调用函数的点之后恢复调用者的执行。
### 活动帧

可以将活动帧视为保存特定函数调用的当前状态的内存块。此状态包括传递给它的任何参数的值以及任何局部变量的值。对于普通函数，活动帧还包括返回地址——从函数返回时将执行转移到的指令的地址——以及用于调用调用函数的活动帧的地址。您可以将这些信息一起视为描述函数调用的关系。 并且它们描述了在该函数完成时应该继续执行哪个函数的哪个调用。对于普通函数，所有活动帧都有严格嵌套的生命周期。这种严格的嵌套允许使用高效的内存分配数据结构来分配和释放每个函数调用的活动帧。这种数据结构通常被称为“堆栈”。当在此堆栈数据结构上分配活动帧时，它通常被称为“堆栈帧”。这种堆栈数据结构非常普遍，以至于大多数（所有？）CPU 架构都有一个专用寄存器来保存指向堆栈顶部的指针（例如，在 X64 中它是 rsp 寄存器）。要为新的活动帧分配空间，只需将该寄存器增加帧大小。要为活动帧释放空间，只需将此寄存器递减帧大小。
### Call调用

当一个函数调用另一个函数时，调用者必须首先为挂起做好准备。

这里说的“挂起”步骤通常涉及将当前保存在 CPU 寄存器中的任何值保存到内存中，以便稍后在函数恢复执行时需要时恢复这些值。根据函数的调用约定，调用者和被调用者可以协调谁保存这些寄存器值，但您仍然可以将它们视为调用操作的一部分。调用者还将传递给被调用函数的任何参数的值存储到新的活动帧中，使得函数可以在其中访问它们。最后，调用者将调用者的恢复点地址写入新的活动帧，并将执行转移到被调用函数的开头。在 X86/X64 架构中，这个最终操作有自己的指令，call指令，它将下一条指令的地址写入堆栈，将堆栈寄存器增加地址的大小，然后跳转到指令中指定的地址操作数。
### Return调用

当函数通过 return 语句返回时，函数首先将返回值（如果有）存储在调用者可以访问的地方。这可能在调用者的活动帧或被调用函数的活动帧中（对于跨越两个活动帧之间边界的参数和返回值，区别可能会有点模糊）。

然后该函数通过以下方式破坏活动帧：
- 在返回点销毁范围内的任何局部变量；
- 销毁任何参数对象；
- 释放活动帧使用的内存；
- 最后，它通过以下方式恢复调用者的执行：
  
  > 通过将堆栈寄存器设置为指向调用者的活动帧并恢复可能已被函数破坏的任何寄存器来恢复调用者的活动帧。
  > 跳转到在call调用期间存储的caller的恢复点。
  > `请注意，与call调用一样，某些调用约定可能会将return的职责拆分为调用者和被调用者函数的指令。`
## Coroutines

协程通过将call和return中执行的一些步骤分成三个额外的操作来概括函数的操作：Suspend、Resume和Destory。
- Suspend 操作在函数内的当前点暂停协程的执行，并将执行转移回调用者或恢复者，而不破坏活动帧。在协程执行暂停后，暂停点范围内的任何对象都保持活动状态。
  
  请注意，就像函数的 Return 操作一样，协程只能在协程本身的明确定义的挂起点处挂起。
- Resume 操作在挂起的点恢复挂起的协程的执行。这将重新激活协程的活动帧。
- Destroy 操作在不恢复执行协程的情况下销毁活动帧。在挂起点范围内的任何对象都将被销毁。用于存储活动帧的内存被释放。
### Coroutine 活动帧

由于协程可以在不破坏活动帧的情况下暂停，我们不能再保证活动帧的生命周期是严格嵌套的。这意味着活动帧通常不能使用堆栈数据结构分配，因此可能需要存储在堆上。C++ Coroutines TS 中有一些规定，如果编译器能够证明协程的生命周期确实严格嵌套在调用者的生命周期内，则允许从调用者的活动帧中为协程框架分配内存。对于协程，活动帧的某些部分需要在协程暂停期间保留，而有些部分只需要在协程执行时保留。例如，具有不跨越任何协程挂起点的范围的变量的生命周期可能会存储在堆栈中。你可以在逻辑上将协程的活动帧视为由两部分组成：“协程框架”和“堆栈框架”。“协程框架”包含协程活动帧的一部分，该框架在协程挂起时持续存在，而“堆栈框架”部分仅在协程执行时存在，并在协程挂起并将执行转移回caller/resumer时被释放。
### 挂起操作

> 协程的 Suspend 操作允许协程在函数中间暂停执行，并将执行转移回协程的调用者或恢复者。

协程主体中的某些点被指定为暂停点。在 C++ Coroutines TS 中，这些挂起点通过使用 co_await 或 co_yield 关键字来标识。当协程遇到这些挂起点之一时，它首先通过以下方式准备协程以恢复：
- 确保将寄存器中保存的任何值写入协程帧
- 向协程帧写入一个值，该值指示协程正在挂起的挂起点。这允许后续的 Resume 操作知道在哪里恢复协程的执行，或者后续的 Destroy 知道哪些值在范围内并且需要销毁。
  
  一旦该coroutine为resume做了准备，该coroutine程序就被视为“suspended”状态。然后，在执行被转回给caller/resumer之前，该coroutine有机会执行一些额外的逻辑。这个额外的逻辑被赋予了对该coroutine的句柄的访问权，该句柄可用于以后恢复或销毁该框架。这种在coroutine进入“suspended”状态后执行逻辑的能力允许coroutine被调度resume，而不需要同步，否则，如果coroutine在进入“suspended”状态前被调度resume，由于coroutine的resume和 suspend可能会发生竞争，就需要同步。下文中会更详细地讨论这个问题。最后，该coroutine可以选择立即恢复/继续执行其本身，也可以选择将执行转回给caller/resumer。如果执行被转移到caller/resumer那，那么coroutine的激活帧的堆栈帧部分被释放并从堆栈中弹出。
### 恢复操作

> 可以在当前处于“suspended”状态的协程上执行恢复操作。

当一个函数想要恢复一个coroutine时，它需要有效地 "调用 "到该函数的一个特定调用的中间。resumer识别要恢复的特定调用的方法是在提供给相应的Suspend操作的`协程帧句柄`上调用void resume()方法。就像正常的函数调用一样，对 resume() 的调用将分配一个新的堆栈帧，并在将执行转移到函数之前将调用者的返回地址存储在堆栈帧中。然而，它不是将执行转移到函数的开始，而是将执行转移到函数中最后被中止的位置。它通过从循环框架中加载恢复点并跳转到该点来实现这一目的。当该coroutine下一次暂停或运行到完成时，对resume()的调用将返回并恢复调用函数的执行。
### 销毁操作

> Destroy 操作在不恢复协程执行的情况下销毁协程框架。
>
> `此操作只能在挂起的协程上执行。`

Destroy 操作的行为与 Resume 操作非常相似，因为它重新激活协程的活动帧，包括分配新的堆栈帧并存储 Destroy 操作的调用者的返回地址。但是，它不是在最后一个挂起点将执行转移到协程主体，而是将执行转移到另一个代码路径，该路径在挂起点调用范围内所有局部变量的析构函数，然后由协程帧释放。与 Resume 操作类似，Destroy操作通过在相应的Suspend操作期间提供的协程帧句柄上调用void destroy()方法来识别要销毁的特定活动帧。
### 协程调用操作

协程的调用操作与普通函数的调用操作非常相似。 事实上，从调用者的角度来看并没有什么区别。然而不是在函数运行完成时才返回调用者的执行，而是使用协程，调用操作将在协程到达其第一个挂起点时恢复调用者的执行。在协程上执行调用操作时，调用者分配一个新的堆栈帧，将参数、返回地址写入堆栈帧并将执行转移到协程。 这与调用普通函数一致。协程做的第一件事是在堆上分配一个协程帧并将参数从堆栈帧复制/移动到协程帧中，以保证参数的生命周期的到延长。
### 协程返回操作

Coroutine的返回操作与普通函数的返回操作有些不同。当coroutine执行一个返回语句（co_return）操作时，它将返回值存储在某个地方（具体存储位置可以由coroutine自定义），然后解构任何范围内的局部变量。然后，coroutine有机会在将执行转回给caller/resumer之前执行一些额外的逻辑。这个额外的逻辑可能会执行一些操作来发布返回值，或者它可能会恢复另一个正在等待结果的coroutine，都是完全可定制。然后coroutine执行Suspend操作（保持协程帧的存活）或Destroy操作（销毁协程帧）。最终，按照Suspend/Destroy操作的语义，执行被转回给调用者/保留者，从堆栈中弹出激活帧的堆栈帧组件。

> 需要注意的是，传递给Return操作的返回值与从Call操作返回的返回值不一样，因为返回操作可能在调用者从最初的Call操作恢复后很久才被执行。
## 调用过程图释

为了帮助将这些概念更容易理解，通过一个简单的例子来说明当一个协程被调用、挂起并随后被恢复时会发生什么。

假设我们有一个函数（或协程） f() 调用协程 x(int a)。

在通话之前，情况看起来有点像这样：

```
STACK                     REGISTERS               HEAP

                        +------+
+---------------+ <------ | rsp  |
|  f()          |         +------+
+---------------+
| ...           |
|               |
```

然后当调用 x(42) 时，它首先为 x() 创建一个堆栈帧，就像普通函数一样。

```
STACK                     REGISTERS               HEAP
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

然后，一旦协程 x() 为堆上的协程框架分配了内存，并将参数值复制/移动到协程框架中，得到如下图所示的结果。 请注意，编译器通常会将协程帧的地址保存在堆栈指针的单独寄存器中（例如，MSVC 将其存储在 rbp 寄存器中）。

```
STACK                     REGISTERS               HEAP
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

如果协程 x() 然后调用另一个普通函数 g() 它将看起来像这样。

```
STACK                     REGISTERS               HEAP
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

当 g() 返回时，它将破坏其活动帧并恢复 x() 的活动帧。 假设我们将 g() 的返回值保存在存储在协程框架中的局部变量 b 中。

```
STACK                     REGISTERS               HEAP
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

如果 x() 现在到达暂停点并暂停执行而不破坏其活动帧，则执行返回到 f()。这导致 x() 的堆栈帧部分从堆栈中弹出，同时将协程帧留在堆上。 当协程第一次挂起时，会向调用者返回一个返回值。 此返回值通常包含一个句柄，指向已挂起的协程框架，可用于稍后恢复它。 当 x() 挂起时，它还将 x() 的恢复点的地址存储在协程帧中（将其称为恢复点的 RP）。

```
STACK                     REGISTERS               HEAP
                                      +----> +-----------+
                        +------+      |      |  x()      |
+----------------+ <----- | rsp  |      |      | a =  42   |
|  f()           |        +------+      |      | b = 789   |
| handle     ----|---+    | rbp  |      |      | RP=x()+99 |
| ...            |   |    +------+      |      +-----------+
|                |   |                  |
|                |   +------------------+
```

这个句柄现在可以作为函数之间的正常值传递。 在稍后的某个时间点，可能来自不同的调用堆栈甚至在不同的线程上，某些东西（例如，h()）将决定恢复执行该协程。 例如，当异步 I/O 操作完成时。恢复协程的函数调用 void resume(handle) 函数来恢复协程的执行。 对于调用者来说，这看起来就像对带有单个参数的 void 返回函数的任何其他正常调用一样。这将创建一个新的堆栈帧，该堆栈帧记录调用者对 resume() 的返回地址，通过将其地址加载到寄存器中来激活协程帧，并在存储在协程中的恢复点恢复 x() 的执行。

```
STACK                     REGISTERS               HEAP
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
## 具体机制

> 标准库中的coroutine带来了什么：
>
> 三个新的语言关键字：co_await、co_yield和co_return
> std::experimental命名空间中的几个新类型：
>
> - `coroutine_handle<P>`
> - `coroutine_traits<Ts...>`
> - `suspend_always`
> - `suspend_never`
>
> 一个通用的机制，库的编写者可以用它来与coroutines交互并定制它们的行为。
> 一种使编写异步代码变得更容易的语言设施。
> C++ Coroutines TS在语言中提供的设施可以被认为是一种用于coroutines的`低级汇编语言`。这些设施很难以安全的方式直接使用，主要是为了让库的编写者用来构建应用程序开发人员可以安全使用的更高级别的抽象。
### 编译器和库的互动

Coroutines标准实际上并没有定义coroutine的语义。它没有定义如何产生返回给调用者的值。它没有定义如何处理传递给co_return语句的返回值，也没有定义如何处理传播到coroutine外的异常，也没有定义该coroutine应该在哪个线程上被恢复。它为库代码指定了一种通用机制，以通过实现符合特定接口的类型来定制coroutine的行为。然后`编译器生成代码`，调用库提供的类型实例上的方法。这种方法类似于库编写者通过定义begin()/end()方法和迭代器类型来定制基于范围的for-loop行为的方式。这种机制允许库的编写者为各种不同的目的定义许多不同类型的coroutine。

有两种接口是由Coroutines标准定义的：Promise接口和Awaitable接口:

1. Promise接口规定了定制coroutine本身行为的方法。编写库的人能够自定义调用coroutine时发生的事情、coroutine返回时发生的事情（无论是通过正常方式还是通过未处理的异常）以及自定义coroutine中任何co_await或co_yield表达式的行为。

2. Awaitable接口指定了控制co_await表达式语义的方法。当一个值被co_await时，代码被翻译成一系列对Awaitable对象上的方法的调用，这些方法允许它指定：是否暂停当前的coroutine，在它暂停后执行一些逻辑以安排coroutine在以后恢复，以及在coroutine恢复后执行一些逻辑以产生co_await表达式的结果。

先看看Awaitable接口。
### Awaiters && Awaitables

co_await操作符是一个新的单数操作符，可以应用于一个值。例如：co_await someValue。co_await操作符只能在一个coroutine的上下文中使用。不过这有点像同义反复，因为任何包含使用co_await操作符的函数体，根据定义，都会被编译为一个coroutine。

> 一个支持co_await操作符的类型被称为Awaitable类型。
>
> 请注意，co_await操作符是否可以应用于一个类型，可能取决于co_await表达式出现的环境。用于coroutine的promise类型可以通过其await_transform方法改变coroutine中co_await表达式的含义。

Awaiter类型是一种实现co_await表达式并且可以被调用的三个特殊方法的类型：await_ready、await_suspend和await_resume。

> 注意，一个类型既可以是Awaitable类型，也可以是Awaiter类型。

当编译器看到co_await <expr>表达式时，根据所涉及的类型，翻译为特定的Awaiters。
#### 获得Awaiter

编译器做的第一件事是生成代码以获得等待值的Awaiter对象。假设等待中的coroutine的Promise对象具有类型P，并且承诺是对当前coroutine的Promise对象的左值引用。

如果promise类型P，有一个名为await_transform的成员，那么<expr>首先被传入对promise.await_transform(<expr>)的调用，以获得Awaitable值，awareitable。否则，如果 promise 类型没有 await_transform 成员，那么直接使用 <expr> 的结果作为 Awaitable 对象 awaitable。然后，如果Awaitable对象（awitable）有一个适用的操作符co_await()重载，那么将调用这个操作符来获得Awaiter对象。否则，对象 awaitable 将被用作等待者对象。

如果我们将这些规则编码到函数get_awaitable()和get_awaiter()中：

```c++
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
if constexpr (has_any_await_transform_member_v<P>)
  return promise.await_transform(static_cast<T&&>(expr));
else
  return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
if constexpr (has_member_operator_co_await_v<Awaitable>)
  return static_cast<Awaitable&&>(awaitable).operator co_await();
else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
  return operator co_await(static_cast<Awaitable&&>(awaitable));
else
  return static_cast<Awaitable&&>(awaitable);
}
```
#### 等待Awaiter

所以，假设已经把把<expr>的结果变成Awaiter对象的逻辑封装在上述函数中，那么co_await <expr>的语义可以翻译如下：

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

当对await_suspend()的调用返回时，await_suspend()的void-returning版本无条件地将执行转移回coroutine的caller/resumer，而bool-returning版本允许awiter对象有条件地立即恢复coroutine，而不返回给caller/resumer。await_suspend()的bool-returning版本在等待者可能开始一个有时可以同步完成的异步操作的情况下很有用。在同步完成的情况下，await_suspend()方法可以返回false，表示应该立即恢复coroutine并继续执行。在<suspend-coroutine>点，编译器会生成一些代码来保存coroutine的当前状态并为恢复做准备。这包括存储<resume-point>的位置，以及将当前保存在寄存器中的任何值溢出到协程帧内存中。在<suspend-coroutine>操作完成后，当前的coroutine被视为暂停。你可以观察到暂停的coroutine的第一个点是在调用await_suspend()的内部。一旦该轮子程序被暂停，它就能够被恢复或销毁。await_suspend()方法的责任是在操作完成后，在未来的某个时间点上为恢复（或销毁）安排该轮子程序。

> 请注意，从await_suspend()中返回false也算作是在当前线程上调度该coroutine立即恢复运行。await_ready()方法的目的是允许你在已知操作将同步完成而不需要暂停的情况下避免<suspend-coroutine>操作的代价。在<return-to-caller-or-resumer>点，执行被转移回调用者或再调用者，弹出本地堆栈帧，但保持协程帧活着。当暂停的 coroutine 最终被恢复时，执行会在 <resume-point> 处恢复，即在 await_resume() 方法被调用以获得操作结果之前立即恢复。await_resume()方法调用的返回值成为co_await表达式的结果。await_resume()方法也可以抛出一个异常，在这种情况下，该异常会传播到co_await表达式之外。

> 请注意，如果异常从 await_suspend() 调用中传播出来，那么该coroutine会自动恢复，并且异常会从 co_await 表达式中传播出来，而无需调用 await_resume()。
### Coroutine句柄

coroutine_handle<P>类型被传递给co_await表达式的await_suspend()调用。这个类型代表了一个对协程帧的非拥有的句柄，可以用来恢复coroutine的执行或销毁协程帧。它也可以用来获取对该coroutine的Promise对象的访问。coroutine_handle类型有以下接口：

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

当实现Awaitable类型时，你将在coroutine_handle上使用的关键方法是.resume()，它应该在操作完成并且你想恢复等待的coroutine的执行时被调用。在一个coroutine_handle上调用.resume()可以在<resum-point>处重新激活一个暂停的coroutine。对.resume()的调用将在该coroutine程序下次到达<return-to-caller-or-resumer>点时返回。.destroy()方法会销毁协程帧，调用任何范围内变量的析构器并释放协程帧所使用的内存。一般来说，你不需要（实际上应该避免）调用.destroy()，除非你是一个实现了coroutinePromise类型的库作者。通常情况下，协程帧将被某种从调用coroutine程序返回的RAII类型所拥有。所以在没有与RAII对象合作的情况下调用.destroy()可能会导致双重破坏的错误。.promise()方法返回一个对coroutine的promise对象的引用。然而，像.destroy()一样，它通常只在你编写coroutine的Promise类型时有用。你应该把coroutine的promise对象视为coroutine的一个内部实现细节。对于大多数正常的 Awaitable 类型，你应该使用 coroutine_handle<void> 作为 await_suspend() 方法的参数类型，而不是 coroutine_handle<Promise>。

coroutine_handle<P>::from_promise(P& promise)函数允许从对coroutine的promise对象的引用中重构coroutine句柄。请注意，你必须确保P的类型与用于协程帧的具体Promise类型完全匹配；当具体的Promise类型是Derived时，试图构造一个coroutine_handle<Base>会导致未定义的行为。.address() / from_address()函数允许将一个coroutine句柄转换为/从一个void*指针。这主要是为了允许作为 "上下文 "参数传递到现有的C风格的API中，所以你可能会发现它在某些情况下对实现Awaitable类型很有用。然而，在大多数情况下，我发现有必要在这个 "上下文 "参数中向回调传递额外的信息，所以我通常最终会将coroutine_handle存储在一个结构中，并在 "上下文 "参数中传递一个指向该结构的指针，而不是使用.address()返回值。
### Synchronisation-free

co_await操作符的一个强大的设计特性是能够在cououtine被暂停后但在执行返回给调用者/resumer之前执行代码。这允许Awaiter对象在已经暂停的cououtine之后启动一个异步操作，将暂停的cououtine_handle传递给该操作，当该操作完成时，它可以安全地恢复（可能在另一个线程上），而无需任何额外的同步要求。例如，通过在await_suspend()中启动一个异步读操作，当coroutine已经暂停时，意味着我们可以在操作完成时恢复coroutine，而不需要任何线程同步来协调启动操作的线程和完成操作的线程。

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

在利用这种方法时要非常小心的一点是，一旦你开始了向其他线程公布coroutine句柄的操作，那么另一个线程可能会在await_suspend()返回之前在另一个线程上恢复coroutine，并可能继续与await_suspend()方法的其他部分同时执行。coroutine恢复时做的第一件事是调用await_resume()来获得结果，然后往往会立即销毁Awaiter对象（即await_suspend()调用的this指针）。然后，该coroutine有可能运行到结束，在await_suspend()返回之前，销毁该coroutine和Promise对象。因此，在 await_suspend() 方法中，一旦该coroutine有可能在另一个线程上并发恢复，你需要确保避免访问这个或该coroutine的 .promise() 对象，因为两者都可能已经被销毁。一般来说，在操作开始后，coroutine被安排恢复时，唯一可以安全访问的是 await_suspend() 内的局部变量。
### 与Stackful Coroutines比较

在许多堆栈式的 coroutine 框架中，一个 coroutine 的暂停操作与另一个 coroutine 的恢复操作相结合，成为一个 "上下文切换 "操作。在这种 "上下文切换 "操作中，通常没有机会在暂停当前的cououtine之后但在将执行转移到另一个cououtine之前执行逻辑。这意味着，如果我们想在堆叠的coroutine之上实现一个类似的异步文件读取操作，那么我们必须在暂停coroutine之前开始操作。因此，操作有可能会在暂停轮子程序之前在另一个线程上完成，并有资格重新开始。这种在另一个线程上完成的操作和暂停的程序之间的潜在竞赛需要某种线程同步来仲裁和决定赢家。可能有一些方法可以解决这个问题，那就是使用一个蹦床上下文，在启动上下文被暂停后代表启动上下文启动操作。然而，这需要额外的基础设施和额外的上下文切换来实现，而且这带来的开销有可能比它试图避免的同步的成本还要高。
### 避免内存分配

异步操作通常需要存储一些每个操作的状态，以跟踪操作的进展。这种状态通常需要在操作的持续时间内持续存在，只有在操作完成后才可以释放。在传统的基于回调的API中，这种状态通常需要在堆上分配，以确保它有适当的寿命。如果你要执行许多操作，你可能需要为每个操作分配和释放这个状态。如果性能是一个问题，那么可以使用一个自定义的分配器，从一个池中分配这些状态对象。然而，当我们使用coroutine时，我们可以通过利用协程帧内的局部变量在coroutine暂停时保持活力这一事实，来避免为操作状态分配堆栈的需要。通过将每个操作状态放在Awaiter对象中，我们可以有效地从协程帧中 "借用 "内存，以便在co_await表达式的持续时间内存储每个操作状态。一旦操作完成，coroutine被恢复，Awaiter对象被销毁，释放出协程帧中的内存供其他局部变量使用。最终，协程帧仍然可以在堆上分配。然而，一旦被分配，一个轮转框架可以被用来执行许多异步操作，只需要那一次堆分配。协程帧就像一种真正的高性能竞技场内存分配器。编译器在编译时计算出所有局部变量所需的总竞技场大小，然后能够以零开销的方式将这些内存分配给局部变量。
### 小例子

> 实现一个简单的同步原语

实现一个基本的可等待的同步原语来展示如何将这些知识付诸实践。一个异步的手动复位事件。这个事件的基本要求是，它需要被多个同时执行的轮子程序所等待，并且在等待时需要暂停等待中的轮子程序，直到某个线程调用.set()方法，这时任何等待中的轮子程序都会恢复。如果某个线程已经调用了.set()，那么该coroutine就应该继续进行而不需要暂停。理想情况下，我们也希望它没有except，不需要堆分配，并且有一个无锁的实现。用法如下所示：

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

让我们首先考虑一下这个事件可能处于的状态。“unset”和 "set"。
- 当它处于 "unset "状态时，有一个（可能是空的）等待中的程序列表，这些程序在等待它成为 "设定"。
- 当它处于 "set "状态时，不会有任何等待的程序，因为在这个状态下共同等待事件的程序可以继续进行而不需要暂停。
  
  > 这个状态实际上可以用一个std::atomic<void*>来表示。为 "set "状态保留一个特殊的指针值。在这种情况下，我们将使用事件的这个指针，因为我们知道它不可能和任何一个列表项的地址相同。
  > 否则，事件就处于 "未设置 "状态，其值是一个指向等待coroutine结构的单链表头的指针。我们可以通过将节点存储在放置在协程帧内的'awaiter'对象中，来避免额外的调用来分配堆上的链接列表的节点。
  
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
  
  这里我们有一个相当直接和简单的接口。在这一点上，主要需要注意的是它有一个操作者co_await()方法，该方法返回一个尚未定义的类型，即awiter。
  
  1. 首先，它需要知道它要等待哪个async_manual_reset_event对象，所以它需要一个对该事件的引用和一个构造函数来初始化它。
  2. 它还需要充当等待者值的链接列表中的一个节点，所以它需要持有一个指向列表中下一个等待者对象的指针。
  3. 它还需要存储正在执行co_await表达式的等待中的coroutine的coroutine_handle，这样当事件变成 "set "时就可以恢复该coroutine。我们不关心该coroutine的Promise类型是什么，所以我们只用一个coroutine_handle<>（这是coroutine_handle<void>的简写）。
  4. 最后，它需要实现Awaiter接口，所以它需要三个特殊的方法：await_ready、await_suspend和await_resume。我们不需要从co_await表达式中返回一个值，所以await_resume可以返回void。
  
  一旦我们把所有这些放在一起，awiter的基本类接口看起来是这样的：
  
  ```c++
  struct async_manual_reset_event::awaiter
  {
  awaiter(const async_manual_reset_event& event) noexcept
  : m_event(event)
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
  
  现在，当我们co_await一个事件时，如果事件已经被设置，我们不希望等待中的coroutine暂停工作。所以我们可以定义await_ready()，如果事件已经被设置，则返回true。
  
  ```c++
  bool async_manual_reset_event::awaiter::await_ready() const noexcept
  {
  return m_event.is_set();
  }
  ```
  
  接下来，让我们看一下await_suspend()方法。这通常是一个可等待类型中最神奇的地方。首先，它需要把等待中的coroutine的句柄藏在m_awaitingCoroutine成员中，以便事件以后可以调用.resume()。然后，一旦我们完成了这些，我们就需要尝试将等待者原子化地排队到等待者的链接列表中。如果我们成功地排队，那么我们就返回true，以表明我们不想立即恢复这个coroutine，否则，如果我们发现事件同时被改变为'set'状态，那么我们就返回false，以表明这个coroutine应该被立即恢复。
  
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
  
  注意，在加载旧状态时，我们使用'acquisition'内存顺序，这样如果我们读取特殊的'set'值，那么我们就可以看到在调用'set()'之前发生的写操作。如果比较交换成功，我们需要 "release "语义，这样后续调用'set()'就可以看到我们对m_awaitingCoroutine的写入和对coroutine状态的先前写入。现在我们已经定义了等待者类型，让我们回头看看async_manual_reset_event方法的实现。首先，构造函数。它需要初始化为空的等待者列表的 "unset"状态（即nullptr）或者初始化为 "set"状态（即this）。
  
  ```c++
  async_manual_reset_event::async_manual_reset_event(
  bool initiallySet) noexcept
  : m_state(initiallySet ? this : nullptr)
  {}
  ```
  
  接下来，is_set()方法是非常直接的--如果它有特殊的值this，它就是“set”。
  
  ```c++
  bool async_manual_reset_event::is_set() const noexcept
  {
  return m_state.load(std::memory_order_acquire) == this;
  }
  ```
  
  接下来是reset()方法。如果它处于 "set "状态，我们就想过渡到空列表的 "unset "状态，否则就保持原样。
  
  ```c++
  void async_manual_reset_event::reset() noexcept
  {
  void* oldValue = this;
  m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
  }
  ```
  
  通过set()方法，我们想通过将当前状态与特殊的 "set "值（this）交换来过渡到 "set "状态，然后检查旧值是什么。如果有任何等待中的程序，那么我们要在返回之前依次恢复每个程序。
  
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
  
  最后，我们需要实现操作者co_await()方法。这只是需要构造一个等待者对象。
  
  ```c++
  async_manual_reset_event::awaiter
  async_manual_reset_event::operator co_await() const noexcept
  {
  return awaiter{ *this };
  }
  ```
### Promise对象

> Promise对象通过实现在执行coroutine过程中的特定点上调用的方法，定义并控制coroutine本身的行为。

在某些使用情况下，coroutine promise对象的作用确实类似于 std::promise 对中的 std::future 部分，但在其他使用情况下，这种类比就有些捉襟见肘。把coroutine的promise对象看作是一个 "coroutine状态控制器 "对象可能更容易，它控制着coroutine的行为，可以用来跟踪其状态。
每次调用coroutine，都会在协程帧内构建一个Promise对象的实例。编译器会在执行coroutine的关键点上生成对Promise对象的某些方法的调用。在下面的例子中，假设为coroutine的特定调用在协程帧中创建的Promise对象。当你写一个具有<body-statements>的coroutine函数，其中包含一个coroutine关键字（co_return、co_await、co_yield），那么coroutine的主体会被转化为如下：。

```c++
{
co_await promise.initial_suspend();
try
{
  <body-statements>
}
catch (...)
{
  promise.unhandled_exception();
}
FinalSuspend:
co_await promise.final_suspend();
}
```

当调用一个coroutine函数时，在执行coroutine主体的源码之前会有一些步骤，这些步骤与普通函数有些不同。

1. 使用操作符new（可选）分配一个协程帧。

2. 将任何函数参数复制到协程帧中。

3. 调用promise对象的构造函数，类型为P。

4. 调用promise.get_return_object()方法，以获得结果，在coroutine第一次暂停时返回给调用者。将结果保存为一个局部变量。

5. 调用promise.initial_suspend()方法并共同等待结果。

6. 当co_await promise.initial_suspend()表达式恢复时（无论是立即还是异步），然后coroutine开始执行你写的coroutine主体语句。

 `当执行到co_return语句时，会执行一些额外的步骤。`

1. 调用 promise.return_void() 或 promise.return_value(<expr>)

2. 按照创建的相反顺序销毁所有具有自动存储期限的变量。

3. 调用 promise.final_suspend() 并 co_await 结果。

 `如果相反，由于一个未处理的异常，执行离开了<body-statements>，那么。`

捕获异常并在捕获块中调用promise.unhandled_exception()。调用 promise.final_suspend() 并共同等待结果。一旦执行扩展到coroutine主体之外，协程帧就会被销毁。销毁该协程帧包括几个步骤:

1. 调用Promise对象的析构器。
2. 调用函数参数副本的析构器。
3. 调用操作者delete来释放被协程帧使用的内存（可选）。
4. 将执行转回给调用者/保留者。
 当执行首次到达co_await表达式内的<return-to-caller-or-resumer>点时，或者如果coroutine运行到完成时没有到达<return-to-caller-or-resumer>点，那么coroutine要么被暂停，要么被销毁，之前从调用promise.get_return_object()返回的返回对象将返回给coroutine的调用者。
#### 分配协程帧

首先，编译器产生一个对操作符new的调用，以便为该协程帧分配内存。

如果Promise类型P定义了一个自定义的操作符new方法，那么它就会被调用，否则就调用全局操作符new。

这里有几件重要的事情要注意。

传递给operator new的大小不是sizeof(P)，而是整个协程帧的大小，并且由编译器根据参数的数量和大小、promise对象的大小、局部变量的数量和大小以及管理coroutine状态所需的其他编译器特定存储来自动决定。

在下列情况下，编译器可以自由地省略对运算符new的调用，作为一种优化。

1. 它能够判断出协程帧的寿命是严格嵌套在调用者的寿命内的；并且

2. 编译器可以看到在调用地点所需的循环框架的大小。

 > 在这些情况下，编译器可以在调用者的活动帧中（无论是在堆栈帧还是协程帧部分）为协程帧分配存储。

Coroutines标准还没有指定任何保证分配的精确性的情况，所以你仍然需要编写代码，就好像协程帧的分配可能会因为std::bad_alloc而失败。这也意味着，你通常不应该将一个coroutine函数声明为noexcept，除非你可以接受在coroutine无法为协程帧分配内存时调用std::terminate()。可以用来代替异常来处理分配协程帧的失败。当在不允许出现异常的环境中操作时，这可能是必要的，例如嵌入式环境或高性能环境，在这些环境中，异常的开销是不能被容忍的。如果Promise类型提供了一个静态的P::get_return_object_on_allocation_failure()成员函数，那么编译器将生成一个对操作者new(size_t, nothrow_t)重载的调用来代替。如果该调用返回nullptr，那么该coroutine将立即调用P::get_return_object_on_allocation_failure()并将结果返回给该coroutine的调用者，而不是抛出一个异常。

`定制协程帧内存分配`

你的promise类型可以定义一个操作符new()的重载，如果编译器需要为使用你的promise类型的协程帧分配内存，它将被调用而不是全局范围的操作符new。

比如说。

```c++
struct my_promise_type
{
void* operator new(std::size_t size)
{
  void* ptr = my_custom_allocate(size);
  if (!ptr) throw std::bad_alloc{};
  return ptr;
}

void operator delete(void* ptr, std::size_t size)
{
  my_custom_free(ptr, size);
}

...
};
```

"但是自定义分配器呢？"，我听到你在问。

你也可以提供一个P::operator new()的重载，该重载需要额外的参数，如果能找到合适的重载，就会用lvalue引用coroutine函数的参数来调用。这可以用来连接operator new来调用分配器上的allocate()方法，该分配器被作为参数传递给coroutine函数。

你需要做一些额外的工作，在被分配的内存中制作一个分配器的副本，这样你就可以在相应的操作符删除的调用中引用它，因为参数没有被传递给相应的操作符删除的调用。这是因为参数被存储在coroutine-frame中，所以在调用operator delete时，它们已经被销毁了。

例如，你可以实现运算符new，使其在协程帧之后分配额外的空间，并使用该空间来存储分配器的副本，该副本可用于释放协程帧的内存。

比如说。

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

为了将自定义的my_promise_type挂在传递std::allocator_arg作为第一个参数的coroutines上，你需要将coroutine_traits类特殊化（更多细节见下面关于coroutine_traits的部分）。

例如。

```c++
namespace std::experimental
{
template<typename ALLOCATOR, typename... ARGS>
struct coroutine_traits<my_return_type, std::allocator_arg_t, ALLOCATOR, ARGS...>
{
  using promise_type = my_promise_type<ALLOCATOR>;
};
}
```

请注意，即使你为一个coroutine定制了内存分配策略，编译器仍然允许省略对内存分配器的调用。
#### 复制参数到协程帧

Coroutine需要将原始调用者传递给coroutine函数的任何参数复制到协程帧中，以便它们在中止后仍然有效。
- 如果参数是通过值传递给coroutine的，那么这些参数将通过调用类型的移动构造器复制到协程帧中。
- 如果参数是通过引用（无论是lvalue还是rvalue）传递给coroutine的，那么只有引用被复制到协程帧中，而不是它们指向的值。
  
  > 请注意，对于具有琐碎的析构器的类型，如果参数在coroutine中可到达的<return-to-caller-or-resumer>点之后从未被引用，编译器可以自由地忽略参数的拷贝。
  
  当以引用方式将参数传入coroutine时，会有很多麻烦，因为你不一定要依靠引用在coroutine的生命周期内保持有效。许多用于普通函数的常用技术，如完美转发和通用引用，如果用于coroutine，会导致代码出现未定义的行为。如果任何一个参数复制/移动构造器抛出了一个异常，那么任何已经构造好的参数都会被析构，协程帧被释放，并且异常会传回给调用者。
#### 构造Promise对象

一旦所有的参数都被复制到协程帧中，coroutine就会构造Promise对象。之所以在构造Promise对象之前复制参数，是为了让Promise对象在其构造函数中能够访问复制后的参数。首先，编译器检查是否有一个 promise 构造函数的重载，它可以接受对每个被复制参数的左值引用。如果编译器找到这样的重载，那么编译器就会生成对该构造函数重载的调用。如果没有找到这样的重载，那么编译器就会退回到生成对Promise类型的默认构造函数的调用。

> 请注意，承诺构造函数 "偷看 "参数的能力是Coroutines相对较新的变化，在Jacksonville 2018会议上被N4723采纳。该提案见P0914R1。因此，它可能不被一些旧版本的Clang或MSVC所支持。

如果Promise构造抛出了一个异常，那么在异常传播到调用者之前，参数副本会被析构，并且在堆栈free间释放了协程帧。
#### 获取返回对象

coroutine对Promise对象做的第一件事是通过调用promise.get_return_object()来获得return-object。return-object是在coroutine第一次暂停时或运行完成后返回给coroutine函数的调用者的值，而执行则返回给调用者。

你可以认为控制流是这样的：

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

> 请注意，需要在启动coroutine体之前获得返回对象，因为协程帧（以及Promise对象）可能会在调用coroutine_handle::resume()返回之前被销毁，无论是在这个线程还是可能在另一个线程，因此在开始执行coroutine体之后调用get_return_object()是不安全的。
#### 最初的挂起点

一旦协程帧被初始化并且获得了return对象，coroutine执行的下一件事就是执行语句co_await promise.initial_suspend();。这使得promise_type的作者可以控制，在执行源代码中出现的coroutine主体之前，coroutine应该暂停，还是立即开始执行coroutine主体。如果该程序在初始暂停点暂停，那么以后可以通过在该程序的coroutine_handle上调用resume()或destroy()，在你选择的时间恢复或销毁它。co_await promise.initial_suspend()表达式的结果被丢弃，所以实现通常应该从awaiter的await_resume()方法返回void。

> 值得注意的是，这个语句存在于保证coroutine其余部分的try/catch块之外。这意味着在<return-to-caller-or-resumer>之前从co_await promise.initial_suspend()中抛出的任何异常都会在销毁协程帧和返回对象之后被抛回coroutine的调用者。

如果你的返回对象有RAII语义，在销毁时破坏了协程帧，请注意这一点。如果是这种情况，那么你要确保co_await promise.initial_suspend()是noexcept，以避免协程帧的double-free。

> 请注意，有人提议调整语义，使co_await promise.initial_suspend()表达式的全部或部分位于coroutine-body的try/catch块内，所以这里的确切语义可能会在coroutine最终确定之前发生变化。
> 对于许多类型的coroutine，initial_suspend()方法要么返回std::experimental::suspend_always（如果操作是慢速启动的），要么返回std::experimental::suspend_never（如果操作是急速启动的），它们都是noexcept awaitable，所以这通常不是一个问题。
#### 返回给caller

当coroutine到达它的第一个<return-to-caller-or-resumer>点（或者如果没有到达这个点，那么当coroutine的执行完成时），从get_return_object()调用返回的return对象将返回给coroutine的调用者。

> 请注意，返回对象的类型不需要和coroutine函数的返回类型相同。如果有必要的话，将执行从返回对象到coroutine的返回类型的隐式转换。
>
> 请注意，Clang的coroutines实现（从5.0开始）推迟执行这种转换，直到从coroutine调用返回对象，而MSVC从2017年更新3开始的实现在调用get_return_object()后立即执行转换。虽然Coroutines TS没有明确说明预期的行为，但我相信MSVC已经计划改变他们的实现，使其行为更像Clang的，因为这可以实现一些有趣的用例。
#### 使用co_return从coroutine返回

当coroutine到达一个co_return语句时，它被翻译成对promise.return_void()或promise.return_value(<expr>)的调用，然后是一个goto FinalSuspend;。

翻译的规则如下。

> - co_return。
>   -> promise.return_void()。
> - co_return <expr>;
>   -> <expr>; promise.return_void(); 如果<expr>的类型为void
>   --> promise.return_value(<expr>); 如果<expr>不具有void的类型

随后的goto FinalSuspend会使所有具有自动存储期限的局部变量在然后评估co_await promise.final_suspend();之前以相反的构造顺序被销毁。

> 请注意，如果执行从一个没有co_return语句的coroutine末端运行，那么这就相当于在函数体的末端有一个co_return。在这种情况下，如果promise_type没有return_void()方法，那么该行为是未定义的。

如果<expr>的评估或对promise.return_void()或promise.return_value()的调用抛出了一个异常，那么这个异常仍然会传播到promise.unhandled_exception()。
#### 处理传播出coroutine的异常

如果一个异常传出了coroutine主体，那么这个异常就会被捕获，并在catch块中调用promise.unhandled_exception()方法。这个方法的实现通常会调用std::current_exception()来捕获异常的副本，并将其存储起来，以便以后在不同的环境中重新抛出。另外，实现者也可以通过执行throw语句立即重新抛出异常。然而，这样做会导致协程帧立即被破坏，并且异常会传播到调用者/响应者那里。这可能会给一些假设/要求调用coroutine_handle::resume()为noexcept的抽象带来问题，所以一般来说，你应该只在完全控制谁/什么人调用resume()时使用这种方法。

> 请注意，如果调用unhandled_exception()重新抛出异常（或者说，如果try-block之外的任何逻辑抛出异常），当前Coroutines的措辞有点不明确。

目前对措辞的解释是，如果控制权退出coroutine，无论是通过从co_await promise.initial_suspend()、promise.unhandled_exception()或co_await promise.final_suspend()传播出来的异常，还是通过co_await p.final_suspend()同步完成的coroutine运行到完成，那么在执行返回给caller/resumer之前，协程帧会自动销毁。然而这种解释有其自身的问题，不是很明确。
#### 最后的挂起点

一旦执行退出coroutine主体的定义部分，并且通过调用return_void()、return_value()或unhandled_exception()捕获结果，以及任何局部变量被析构，coroutine有机会在执行返回给调用者/resumer之前执行一些额外的逻辑。比如传播结果、发出完成信号或恢复额外的coroutine。它还允许coroutine在coroutine执行完成和协程帧被销毁之前，有选择地立即暂停。

> 请注意，resume()一个在final_suspend点被暂停的cououtine是未定义的行为，能对在此暂停的程序做的唯一事情就是调用destory()它。

这种限制的理由是，由于减少了需要由cououtine表示的暂停状态的数量，以及可能减少所需的分支数量，这为编译器提供了一些优化的机会。

> 请注意，虽然可以让一个coroutine不在final_suspend点暂停，但建议你在结构上让你的coroutine尽可能在final_suspend点暂停。这是因为这将迫使你从coroutine外部调用.destroy()（通常是从一些RAII对象的析构器），这使得编译器更容易确定协程帧的生命周期范围何时嵌套在调用者内部。这反过来又使得编译器更有可能避免对协程帧的内存分配。
#### 编译器如何选择Promise类型

因此，现在让我们看看编译器是如何确定在一个给定的coroutine中使用什么类型的promise对象的。Promise对象的类型是通过使用std::experimental::coroutine_traits类从coroutine的签名中确定的。

如果你有一个coroutine函数的签名：

```c++
task<float> foo(std::string x, bool flag);
```

编译器将通过把返回类型和参数类型作为模板参数传递给coroutine_traits来推断coroutine的Promise类型：

```c++
typename coroutine_traits<task<float>, std::string, bool>::promise_type;
```

如果该函数是一个非静态成员函数，那么类的类型将作为第二个模板参数传递给coroutine_traits。注意，如果你的方法是为右值引用重载的，那么第二个模板参数将是一个右值引用。例如：

```c++
task<void> my_class::method1(int x) const;
task<foo> my_class::method2() &&;
```

那么编译器将使用以下Promise类型

```c++
// method1 promise type
typename coroutine_traits<task<void>, const my_class&, int>::promise_type;

// method2 promise type
typename coroutine_traits<task<foo>, my_class&&>::promise_type;
```

coroutine_traits模板的默认定义是通过寻找定义在返回类型上的嵌套promise_type类型定义：

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

因此，对于控制的coroutine返回类型，你可以在你的类中定义一个嵌套的promise_type，让编译器使用该类型作为返回你的类的coroutine对象的类型。比如：

```c++
template<typename T>
struct task
{
using promise_type = task_promise<T>;
...
};
```

然而对无法控制的coroutine返回类型，可以对coroutine_traits进行特定处理，定义要使用的Promise类型，而不需要修改原本的类型。例如，为一个返回std::optional<T>的coroutine定义要使用的Promise类型：

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
#### 识别一个特定的coroutine活动帧

当你调用coroutine函数时，一个协程帧被创建。为了恢复相关的程序或销毁该协程帧，你需要某种方式来识别或引用该特定的协程帧。Coroutines标准库为此提供的机制是coroutine_handle类型。接口如下：

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

可通过两种方式为一个coroutine程序获得一个coroutine_handle。

1. 在co_await表达式中，它被传递给await_suspend()方法。
2. 如果你有一个对该程序的Promise对象的引用，你可以使用coroutine_handle<Promise>::from_promise()来重建其coroutine_handle。等待中的 coroutine_handle 将被传递到 awaiter 的 await_suspend() 方法中，在 coroutine 在 co_await 表达式的 <suspend-point> 处暂停后。你可以把这个coroutine_handle看作是在一个延续传递式的调用中代表coroutine的延续。

注意，coroutine_handle不是RAII对象。你必须手动调用.destroy()来销毁该协程帧并释放其资源。可以把它看作是void*的等价物。这是为了性能的原因：使它成为一个RAII对象会给coroutine增加额外的开销，例如需要引用计数。**一般来说，你应该尽量使用为coroutine提供RAII语义的高级类型，比如那些由cppcoro提供的类型或者编写你自己的高级类型，为你的coroutine类型封装协程帧的生命周期。**
#### 定制co_await的行为

Promise类型可以选择性地定制出现在coroutine主体中的每个co_await表达式的行为。通过简单地在 promise 类型上定义一个名为 await_transform() 的方法，编译器将把出现在 coroutine 主体中的每个 co_await <expr> 转变为 co_await promise.await_transform(<expr>) 。可以启用通常不可等待的等待类型。例如，用于具有 std::optional<T> 返回类型的 coroutine 的 promise 类型可以提供一个 await_transform() 重载，它接受一个 std::optional<U> 并返回一个可等待的类型，该类型要么返回一个 U 类型的值，要么在等待的值包含 nullopt 时暂停 coroutine。

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

允许通过将 await_transform 重载声明为删除，来禁止在某些类型上进行等待。例如，一个用于std::generator<T>返回类型的Promise类型可以声明一个删除的await_transform()模板成员函数，接受任何类型。这基本上禁止了coroutine中co_await的使用：

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

允许调整和改变通常可等待值的行为例如，你可以定义一种coroutine，确保coroutine总是从相关的执行器上的每个co_await表达式中恢复，方法是将可等待值包装在resume_on()操作符中（比如cppcoro::resume_on()）。

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

关于 await_transform() 的最后一句话，重要的是要注意，如果Promise类型定义了任何 await_transform() 成员，那么这将触发编译器将所有 co_await 表达式转换为调用 promise.await_transform() 。这意味着，如果为某些类型定制co_await的行为，你还需要提供一个await_transform()的回退重载，它只是转发参数。
#### 定制co_yield的行为

可以通过Promise类型定制co_yield关键字的行为。如果co_yield关键字出现在一个coroutine中，那么编译器会将表达式co_yield <expr>翻译成表达式co_await promise.yield_value(<expr>)。Promise类型可以通过在Promise对象上定义一个或多个 yield_value() 方法来定制 co_yield 关键字的行为。注意，与 await_transform 不同，如果 promise 类型没有定义 yield_value() 方法，则co_yield 没有默认行为。因此，一个Promise类型需要通过声明删除 await_transform() 显式不允许 co_await，但需要支持 co_yield。带有 yield_value() 方法的Promise类型的典型例子是 generator<T> 类型:

```c++
template<typename T>
class generator_promise
{
T* valuePtr;
public:
...

std::experimental::suspend_always yield_value(T& value) noexcept
{
  // Stash the address of the yielded value and then return an awaitable
  // that will cause the coroutine to suspend at the co_yield expression.
  // Execution will then return from the call to coroutine_handle<>::resume()
  // inside either generator<T>::begin() or generator<T>::iterator::operator++().
  valuePtr = std::addressof(value);
  return {};
}
};
```
## 总结

C++20 的协程设计为无栈协程，相对于有栈协程，省掉了上下文切换开销，只能手动切换，效率更高，也不用管理复杂的寄存器状态，移植性更好，但这同时也导致了不能被非协程函数嵌套调用。

同时引入了 3 个关键字：
- co_yield: 挂起并返回值
- co_await: 挂起
- co_return: 结束协程
  
  当一个函数出现了上面的关键字，则该函数是个协程。
  
  `Promise`
  
  当 caller 调用一个 callee 协程的时候，协程自身的状态信息（形参，局部变量，自带数据，各个阶段点执行点）会被保存在堆上的 Promise 对象中，这也是编译器会在协程里面插入 Promise 相关代码，以及一些执行点。由于 Promise 的大小可以在编译期计算出来，从而避免了内存浪费。而 Promise 对象所有权可由`coroutine_handle` 句柄持有。
  
  `Future`
  
  而 Future 对象主要是与 Promise 对象交互的桥梁，既 caller 与 callee 之间的通信：
- callee 挂起时，将值返回给 caller: yield 语义
- callee 执行结束时，将值返回给 caller: return 语义
- callee 恢复时，caller 将值带给 callee
  
  需要注意的是，这些概念和标准库的 `std::promise/std::future` 不是同一个东西，后者用于做同步用，`std::future`会阻塞等待直到 `std::promise` 提供值，可以看做是条件变量的封装，同样地，和其他语言的 Promise/Future 概念也不一样。
  
  `Awaitable`
  
  如果一个对象是 Awaitable 对象，那么可以用 co_await 操作符去触发该对象的动作 ready/suspend/resume，从而转移、恢复控制权，co_await 细节留到后面在介绍。
  
  。
  
  `Promise/Future 对象`
  
  当一个协程被调用时，会创建 Promise 对象，然后编译器会在各个阶段插入一些代码
  
  ```cpp
  {
  co_await promise.initial_suspend();
  try
  {
    <body-statements>
  }
  catch (...)
  {
    promise.unhandled_exception();
  }
  FinalSuspend:
  co_await promise.final_suspend();
  }
  ```
  
  可以看到一个协程函数，分为如下几个步骤：
  
  1. 从堆上 (operator new) 创建 Promise 对象，保存协程的状态信息
  2. initial_suspend 阶段，用于在执行协程主体 `<body-statements>` 代码前做些事情
  3. `<body-statements>`阶段，执行协程的主体代码
  4. unhandled_exception 阶段，若抛异常，处理异常
  5. final_suspend 阶段，协程结束收尾动作，在这阶段的 `coroutine_handle<Promise>::done` 方法为 true，caller 可以通过这个方法判断协程是否结束，从而不再调用 resume 恢复协程
  
  而协程返回类型则是一个 Future 对象，这一步编译器通过 `Promise::get_return_object()` 来创建 Future 对象。而 Future 对象一般持有 Promise 的句柄：`coroutine_handle<Promise>`，这样 caller 可以通过 Future 与 Promise 交互，从而恢复协程。
  
  而 Promise 对象释放的时间点有两个，避免重复执行，否则会 double free：
- final_suspend 阶段 resume 后
- 调用 `coroutine_handle<Promise>::destroy()` 方法
  
  比较好的做法是在 final_suspend 阶段挂起，这时候就不可 resume 了，在 caller 通过调用 Future 持有的句柄 `destroy()` 方法释放 Promise 对象。
  
  综上，一个 Promise 对象需要实现如下方法：
- initial_suspend: 返回一个 Awaitable 对象
- final_suspend: 返回一个 Awaitable 对象
- get_return_object: 返回一个 Future 对象给 caller
- unhandled_exception: 处理异常
- return_value/return_void: co_return 时返回值给 caller
- yield_value: 挂起时返回值给 caller
  
  再来看看其 `coroutine_handle<Promise>` 句柄编译器提供了哪些主要方法：
- destroy: 销毁 Promise 对象
- from_promise: 静态方法，从 Promise 对象返回其 `coroutine_handle` 句柄
- done: 是否处于 final_suspend 阶段
- promise: 返回 Promise 对象引用
- resume/operator(): 恢复到协程
  
  `Awaitable 对象`
  
  前面提到的 co_await 关键字，其操作的对象其实是 Awaiter 对象，若对象实现如下方法，则说明该对象是 Awaitable 的：
- await_ready
- await_suspend(`coroutine_handle<>`)
- await_resume
  
  那么当执行 `co_await <expr>` 表达式时，编译器会生成如下代码：
  
  ```cpp
  {
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    <suspend-coroutine>
  
    //if await_suspend returns void
    try {
        awaiter.await_suspend(coroutine_handle);
        return_to_the_caller();
    } catch (...) {
        exception = std::current_exception();
        goto resume_point;
    }
    //endif
    //if await_suspend returns bool
    bool await_suspend_result;
    try {
        await_suspend_result = awaiter.await_suspend(coroutine_handle);
    } catch (...) {
        exception = std::current_exception();
        goto resume_point;
    }
    if (not await_suspend_result)
        goto resume_point;
    return_to_the_caller();
    //endif
    //if await_suspend returns another coroutine_handle
    decltype(awaiter.await_suspend(std::declval<coro_handle_t>())) another_coro_handle;
    try {
        another_coro_handle = awaiter.await_suspend(coroutine_handle);
    } catch (...) {
        exception = std::current_exception();
        goto resume_point;
    }
    another_coro_handle.resume();
    return_to_the_caller();
    //endif
  }
  resume_point:
  if(exception)
  std::rethrow_exception(exception);
  "return" awaiter.await_resume();
  }
  ```
  
  也就是：
  
  1. 通过 `<expr>` 拿到 Awaiter 对象
  2. 通过 Awaiter.await_ready()方法判断是否需要挂起，若为 true 则无需挂起
  3. 判断 Awaiter.await_suspend()的返回值类型：
	- void，无返回值，直接挂起返回 caller
	- bool，若为 true，则返挂起返回 caller，否则不挂起，直接 re