---
title: cloudwu/coroutine 源码分析
date: 2022-06-06 19:13
---

## 1 与其它协程库使用对比
这个 C 协程库是云风（cloudwu） 写的，其接口风格与 Lua 协程类似，并且都是非对称 stackful 协程。这个是源代码中的示例：
```c
#include "coroutine.h"
#include <stdio.h>

struct args
{
    int n;
};

static void
foo(struct schedule *S, void *ud)
{
    struct args *arg = ud;
    int start = arg->n;
    int i;
    for (i = 0; i < 5; i++)
    {
        printf("coroutine %d : %d\n", coroutine_running(S), start + i);
        coroutine_yield(S);
    }
}

static void
test(struct schedule *S)
{
    struct args arg1 = {0};
    struct args arg2 = {100};

    int co1 = coroutine_new(S, foo, &arg1);
    int co2 = coroutine_new(S, foo, &arg2);
    printf("main start\n");
    while (coroutine_status(S, co1) && coroutine_status(S, co2))
    {
        coroutine_resume(S, co1);
        coroutine_resume(S, co2);
    }
    printf("main end\n");
}

int main()
{
    struct schedule *S = coroutine_open();
    test(S);
    coroutine_close(S);

    return 0;
}
```
这段代码输出：
```
main start
coroutine 0 : 0
coroutine 1 : 100
coroutine 0 : 1
coroutine 1 : 101
coroutine 0 : 2
coroutine 1 : 102
coroutine 0 : 3
coroutine 1 : 103
coroutine 0 : 4
coroutine 1 : 104
main end
```
与其等价的 Lua 代码为：
```lua
local function foo(args)
    local start = args.n
    for i = 0, 4 do
        print(string.format('coroutine %s : %d', coroutine.running(), start + i))
        coroutine.yield()
    end
end

local function test()
    local arg1 = {n = 0}
    local arg2 = {n = 100}

    local co1 = coroutine.create(foo)
    local co2 = coroutine.create(foo)
    print('main start')
    coroutine.resume(co1, arg1)
    coroutine.resume(co2, arg2)
    while coroutine.status(co1) ~= 'dead' and coroutine.status(co2) ~= 'dead' do
        coroutine.resume(co1)
        coroutine.resume(co2)
    end
    print('main end')
end

test()
```
这段代码输出：
```
main start
coroutine thread: 000001D62BD6B8A8 : 0
coroutine thread: 000001D62BD6BA68 : 100
coroutine thread: 000001D62BD6B8A8 : 1
coroutine thread: 000001D62BD6BA68 : 101
coroutine thread: 000001D62BD6B8A8 : 2
coroutine thread: 000001D62BD6BA68 : 102
coroutine thread: 000001D62BD6B8A8 : 3
coroutine thread: 000001D62BD6BA68 : 103
coroutine thread: 000001D62BD6B8A8 : 4
coroutine thread: 000001D62BD6BA68 : 104
main end
```
与其等价的 C++ 20 代码为：
```cpp
#include <coroutine>
#include <functional>
#include <stdio.h>

struct coroutine_running
{
    bool await_ready() { return false; }
    bool await_suspend(std::coroutine_handle<> h) {
        _addr = h.address();
        return false;
    }
    void* await_resume() {
        return _addr;
    }
    void* _addr;
};

struct return_object : std::coroutine_handle<>
{
    struct promise_type
    {
        return_object get_return_object() {
            return std::coroutine_handle<promise_type>::from_promise(*this);
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };

    return_object(std::coroutine_handle<promise_type> h) : std::coroutine_handle<>(h){}
};

struct args
{
    int n;
};

return_object foo(args* arg)
{
    int start = arg->n;
    for (int i = 0; i < 5; i++)
    {
        printf("coroutine %p : %d\n", co_await coroutine_running{}, start + i);
        co_await std::suspend_always{};
    }
}

int main()
{
    args arg1 = { 0 };
    args arg2 = { 100 };
    auto co1 = foo(&arg1);
    auto co2 = foo(&arg2);
    printf("main start\n");
    while (!co1.done() && !co2.done())
    {
        co1.resume();
        co2.resume();
    }
    co1.destroy();
    co2.destroy();
    printf("main end\n");
}
```
这段代码输出
```
main start
coroutine 0x607000000020 : 0
coroutine 0x607000000090 : 100
coroutine 0x607000000020 : 1
coroutine 0x607000000090 : 101
coroutine 0x607000000020 : 2
coroutine 0x607000000090 : 102
coroutine 0x607000000020 : 3
coroutine 0x607000000090 : 103
coroutine 0x607000000020 : 4
coroutine 0x607000000090 : 104
main end
```
对比三段代码，可以看到 C 版本比 Lua 版本多了 coroutine_open 和 coroutine_close，C 版本需要保存一个全局状态 S。Lua 版本无法在创建协程的时候指定参数，必须在之后的 resume 把参数传递给 foo。C++ 20 版本需要手写第 5~35 行的框架代码才能达到同样的效果。抛开这三段代码的差异，能看到协程库的共性：创建协程（create）、恢复协程（resume）、让出协程（yield）、销毁协程（destroy）。cloudwu/coroutine 的源代码行数非常少，所以以此分析一个协程库如何实现这些机制。

在分析代码之前需要明确协程的一些基本概念，coroutine 的 co 并不是 concurrency，而是 cooperative。协程就是能协作执行的例程（routine）。函数（function）也是例程，但不是协程。协程和函数是类似概念。协程和线程不类似，和线程类似的概念是纤程（Fiber）。关于协程和纤程的区别可以看 [N4024](https://wg21.link/n4024)。协程按能不能在嵌套栈帧中挂起（suspend），分为：stackless 协程 和 stackful 协程。stackless 协程只能在最顶层栈帧挂起，stackful 协程能在任意栈帧挂起。C++ 20 协程是 stackless 协程。所以在上面的代码中，如果 foo 再调用一个函数 bar，bar 就无法使用 co_await。但是 C 版本和 Lua 版本的协程可以这样，所以它们是 stackful。如果 a resume b，则称 a 为 b 的 resumer。协程还可以按控制流（control flow）切换方式分为：对称（symmetric）协程和非对称（asymmetric）协程。非对称协程通过 yield 切换到其 resumer，不需要指定切换到哪个协程。对称协程每次切换都需要指定切换到哪个协程，它可以切换到任意协程。C 版本和 Lua 版本的协程都只支持非对称协程，但是可以通过非对称协程实现对称协程。C++ 20 协程两者都支持。

源代码只有两个文件 coroutine.h 和 coroutine.c。

##  2 coroutine.h

先看 coroutine.h，有 4 个宏定义

| Name | Value |
| ---- | ---- |
| COROUTINE_DEAD | 0 |
| COROUTINE_READY | 1 |
| COROUTINE_RUNNING | 2 |
| COROUTINE_SUSPEND | 3 |

显然，这个 4 个宏定义了协程的四种状态，协程创建完还没执行是 COROUTINE_READY，正在执行是COROUTINE_RUNNING，被挂起是 COROUTINE_SUSPEND，已经结束是 COROUTINE_DEAD。为方面阅读，下面直接用 Ready、Running、Suspend、Dead 指代。

一个 schedule 结构体，保存了用来做协程调度的信息，是一个协程组的全局状态。注意协程并没有调度器，也不需要像进程和线程那样的调度算法。每次协程的切换，切换到哪个协程都是固定的。所有属于同一个 schedule 的协程可以视为一个协程组。schedule 拥有这个协程组的信息。

一个函数指针定义 coroutine_func，这个函数表示协程的执行内容，也就是协程执行的时候会调用这个函数。该函数有一个参数 ud，ud 是 user data 的缩写，user 指代调用接口的程序员，user data 被用来传递数据。Lua 的 userdata 也用于 C 和 Lua 之间数据传递。

两个针对全局状态 S 的操作：coroutine_open 和 coroutine_close

五个针对单个协程的操作：coroutine_new、coroutine_resume、coroutine_status、coroutine_running、coroutine_yield。其中 coroutine_new 也有一个 ud 和 coroutine_func 的参数对应。根据这些状态和操作可以画出协程的状态图。

```mermaid
stateDiagram
    [*] --> Ready : coroutine_new
    Ready --> Running : coroutine_resume
    Running --> Suspend : coroutine_yield
    Suspend --> Running : coroutine_resume
    Running --> Dead : normal finish / coroutine_close
    Dead --> [*]
```

协程从 Running 转变为 Dead，有两种途径，一是正常结束返回，二是调用 coroutine_close 关闭所有协程。两者都会销毁协程。协程被销毁之后，其保存的状态也不存在了，只不过用 Dead 表示其不存在。观察这个状态图，可以和 Lua 的协程状态进行类比，发现 Lua 协程还具有 normal 状态，也就是 main resume a, a resume b，a 是 normal 状态。经过测试，C 版本协程库无法嵌套 resume，a 不能 resume b。resumer 只能是 main routine。也就没有 normal 状态。

## 3 coroutine.c

在这个文件里包含了 `ucontext.h`，说明这个库使用 ucontext 进行 context 切换。查看 ucontext_t 的定义，可以看到它保存了 CPU 的寄存器值和其它状态信息，包括通用寄存器 rax、rbx、rcx、rdx ...，用于浮点数计算的XMM寄存器，中断屏蔽字 sigmask 等。

STACK_SIZE 定义了所有协程栈的容量为 1MB，DEFAULT_COROUTINE 定义了初始的协程数量为 16。注意初始的协程数量为 16，并不代表创建了 16 个协程，只是分配了大小为 16 的协程指针数组。

schedule 结构体里的 statck 是大小为 STACK_SIZE 的字节数组。表示协程组所有协程的工作栈，这个栈是所有协程共享的，它是固定分配的，不会动态扩容。当任意一个协程被恢复，这个协程会使用这个工作栈执行代码，分配栈变量。main 表示 main routine 的 ucontext。nco 表示当前存在的协程数量。cap 是 capacity 的缩写，表示协程指针数组的容量，初始大小是 DEFAULT_COROUTINE。running 表示当前正在执行的协程 id。co 是协程指针的指针，也就是一个协程指针数组。为什么使用协程指针数组，而不直接使用协程数组？因为扩容的时候效率更高。co 扩容的时候，如果使用协程指针数组，挪动的每个格子大小为 `sizeof(struct coroutine *)`。反之，直接使用协程数组，挪动的每个格子大小为 `sizeof(coroutine)`。哪个效率高，一目了然。

coroutine 结构体保存了每个协程的状态信息。func 表示协程的执行内容，由 coroutine_new 的 func 参数指定。ud 表示 func 的参数，由 coroutine_new 的 ud 参数指定。ctx 表示该协程的 ucontext。sch 指向全局状态 S，也就是每个协程可以反向查找到其属于哪个协程组。cap 是该协程的私有栈容量，size 是实际占用大小。status 表示该协程的状态，也就是前面提到的四种状态，但是 status 永远不会被置为 Dead。因为一个协程销毁之后，就会在 S 中的协程指针数组中对应格子置为 NULL。status 没有机会被置为 Dead。stack 表示该协程的私有栈，是动态扩容的。

共享栈是所有协程的工作栈，同一时间有且只有协程正在执行，所有只需要一个工作栈就行了。每个协程都有自己的栈数据，因此每个协程都需要自己私有栈。为什么不直接用私有栈作为工作栈，这样还省去了共享栈和私有栈之间数据复制？因为这里采取的策略是用时间换取空间。假设没有共享栈，所有协程使用自己的私有栈作为工作栈。要保证同协程栈变量地址一致性，也就是一个局部变量在协程切出去和切回来之后的地址是相同的，工作栈无法动态扩容。因为一旦扩容，无法保证这个一致性，user code 可能会出问题。所以工作栈无法动态扩容，所有私有栈大小都为 STACK_SIZE。假设有 1024 个协程都处于 Ready 状态，即使还没执行，也占用了 1 GB 的内存。反之如果使用共享栈作为工作栈，则只占用 1MB 的内存，其余私有栈都是未分配的状态。共享栈保证了同协程栈变量地址一致性，但是无法保证异协程栈变量地址一致性。例如，协程 a 访问协程 b 的一个栈变量地址，访问的是共享栈的地址，user code 无法正常工作。通常来说这种使用场景非常少，也可以将访问栈变量地址改为访问堆变量地址来规避这个问题。

下面从头文件的接口函数入手，逐个进行分析

### 3.1 coroutine_open

该函数为 S 分配内存，初始化其每个成员，为协程指针数组 co 预留 DEFAULT_COROUTINE 个格子，并全部置为 NULL。如果仔细观察，可以发现 stack 并没有进行 memset 操作，是忘记写了吗？其实是故意为之。因为栈内存的初始化应该由 user code 承担，这里没必要进行初始化。main 也没有进行初始化，实际上所有 ucontext_t 结构体都不需要初始化，它们在被使用之前都应该使用 getcontext 赋值。

### 3.2 coroutine_close

该函数销毁所有协程，然后销毁全局状态 S。这里直接遍历了所有的格子，因为现存的协程可能并不是正好占据第 0 ~ nco - 1 个格子。

### 3.3 coroutine_new

该函数分配一个 coroutine 结构体并执行初始化。可以看到，此时协程的状态被置为 Ready。协程的私有栈没有被分配，等到协程真正要被执行的时候再分配。然后把协程指针保存到 S 中，但是 S 的 co 格子数量可能不够，需要动态扩容。这里的扩容策略是系数为 2 的线性增长。这里扩容使用了 realloc 函数，这个函数比使用 malloc + free 效率更高。如果格子数量足够就从第 nco 个格子开始查找空位。如果达到 cap - 1，就回绕从第 0 个格子继续查找。也就是进行循环查找，能够最大化利用 co 的所有空格。从第 nco 个格子开始查找具有一定的效率优势，例如假设现存 10 个协程，也没有协程被销毁，在分配第 11 个协程时，能够直接找到第 10 个格子，它正好是空格子。但是，一旦协程被随机销毁，这种优势会消失，占用的格子会被随机分布，找第 nco 个格子和随机选择一个格子没什么区别。assert(0) 是一个运行期断言，如果走到这里，说明 S 的数据被破坏了，直接执行 abort() 终止程序。

### 3.4 coroutine_resume

该函数挂起当前的函数，恢复指定的目标协程。`assert(S->running == -1);` 说明调用者不能是属于 S 协程组的协程。因为如果调用者是协程，runnning 不可能为 -1，程序会直接退出。`assert(id >=0 && id < S->cap);` 校验 id 是否合法。接下来根据目标协程的状态进行操作。可以看到只能 resume 一个状态为 Ready 或 Suspend 的协程。

可以看到，恢复一个 Ready 协程，使用到了三个 UNIX 函数，分别为 getcontext、makecontext、swapcontext。实际上这组 UNIX 函数有四个，只不过只用了这三个。

```c
#include <ucontext.h>

int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);

void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
int swapcontext(ucontext_t *restrict oucp, const ucontext_t *restrict ucp);
```

getcontext 保存当前的 context 到 ucp。setcontext 用 ucp 设置当前的 context，也就是激活 ucp。这个函数一旦调用成功，就不会返回。这种调用成功不返回的函数，与 longjmp 类似，它也是这样。实际上，用户态 context 切换机制经历了三个发展阶段。第一阶段是 setjmp/longjmp。第二阶段是 sigsetjmp/siglongjmp，它能够控制中断屏蔽字。第三个阶段是 getcontext/setcontext，它能够提供更加完整的控制。

makecontext 可以修改一个 context，它可以修改 ucp 的里面的一些寄存器和状态，使得 ucp 被激活后会调用 func(...) 函数。argc 表示后面的参数个数，后面可以传递任意个数的参数。但是为了可移植性，必须都是 int 类型。代码里面使用的是 uint32_t 与 int 相符。Linux x86_64 的数据模型是 LP64，也就是 long 和 pointer 都是64位。因此，如果要传递指针 S，则需要将 S 拆成两个 32 位的 uintptr_t。当然这段代码也支持 Linux x86，高 32 位都是 0。在调用 makecontext 之前必须设置 ucp->uc_stack 和 ucp->uc_link。ucp->uc_stack 表示 ucp 激活之后，func 函数在哪个栈上执行。因此需要对 uc_stack.ss_sp 和 uc_stack.ss_size 赋值。代码里面直接设置了工作栈 S->stack，和其大小 STACK_SIZE。这里需要注意的是 uc_stack.ss_sp 中的 sp 并不像 rsp 寄存器那样表示 stack pointer，指向栈顶。x86 架构的栈都是从高地址向低地址扩展。如果 ss_sp 表示 stack pointer 栈顶，则应该赋值为 S->stack + STACK_SIZE。实际上，uc_stack.ss_sp 的 sp 表示 starting pointer，表示工作栈的起始地址，也就是 S->stack。当 ucp 被激活后，会根据 CPU 架构自动设置 rsp，所以不必考虑这些。ucp->uc_link 表示 func 函数执行完成之后激活哪个 context。显然，协程执行完成之后，应该继续执行其 resumer，也就是 main routine。注意，代码里的 uc_link 赋值时，S->main 还没有任何有效内容。但是 uc_link 只需要保存 ucontext_t 的指针，所以可以这样做。S->running 赋值为 id。这里协程还没有真正执行，context 也没有切换，只是方便传参，mainfunc 可以通过 S 拿到 id。

swapcontext 并不是交换其两个参数 oucp 和 ucp。而是 oucp、当前 context、ucp 三者之间进行交换。 流程如下图所示：

```mermaid
flowchart RL
	cucp(当前 context)
	ucp -- 2 --> cucp -- 1 --> oucp
	
```

必须按顺序先将当前context 保存到 oucp，再激活 ucp。不然 S->main 的内容还是无效的，就被切换到了 mainfunc。这里使用 swapcontext，而不是分两次调用 getcontext 和 setcontext。假设使用 getcontext + setcontext，context 切换出去，后面再切回来的时候，会回到 getcontext 刚刚执行之后，又会再次执行 setcontext，这显然不行。使用 swapcontext 可以规避这个问题，当 context 切出去再切回来时，会正好回到 swapcontext 刚刚执行完毕。

为什么切换到 mainfunc，而不是直接切换到 C->func ？因为当 C->func 执行完成之后，需要销毁协程对象 C，也需要修改 S 的信息。所以使用 mainfunc 对 C->func 进行了包装。可以看到 mainfunc 执行了 C->func 之后确实进行了一些清理操作。所以协程执行完毕之后不必手动进行 destroy 操作，会自动释放内存。如果想在 user code 里面提前销毁被挂起的协程怎么办？作者预留了 _co_delete 接口，它没有出现在头文件中，但也没有用 static 修饰。也就是说，可以自己在 user code 写一遍 _co_delete 的声明，再手动清理。

恢复一个 Suspend 协程则相对简单得多。C->ctx 早已保存有效的信息。不需要再次设置，也不需要调用 makecontext，只需要 restore 协程私有栈的内容到工作栈 S->stack。由于私有栈仅保存 user code 中已经被占用的内存，所以只复制 C->size 大小的内存。这里默认栈使用从高地址向低地址的扩展方式，也就是该代码只适用于这种栈扩展方式的架构，如 x86 架构。私有栈内容复制到共享栈的地址对齐如下图所示，栈底的 S->stack + STACK_SIZE 和 C->stack + size 都是实际占用内存的下一个地址。

```txt
                                         S->stack       C->stack
High Addr    S->stack + STACK_SIZE ---> +-------+       +-------+ <-- C->stack + size
    |                                   |       |       |       |
    |                                   |       |       |       |
    |        S->stack + STACK_SIZE      |       |       |       |
    |              - C->size       ---> |-------|<----- +-------+ <-- C->stack
    |                                   |       |
    V                                   |       |
Low Addr                  S->stack ---> +-------+
```

### 3.5 coroutine_yield

执行 coroutine_yield 会挂起当前协程，并且切换到其 resumer 继续执行。如果简单地使用 swapcontext，会出现问题。如协程 a 挂起，main routine 再恢复协程 b 执行一些操作，b 再挂起，main routine 再恢复 a。这时 a 的栈变量可能已经被 b 修改了。这是因为 a 和 b 共享一个工作栈。所以在 swapcontext 之前需要将当前协程的工作栈内容 store 到其私有栈。在调用 `_save_stack` 函数之前有一个断言，`assert((char *)&C > S->stack);` 这个断言只能用来校验 &C 是否超过 S->stack 的下界。虽然 C 是一个指针，但是也是一个栈变量，其地址和当前工作栈的栈顶 S->stack 进行比较。如果 &C > S->stack 则表示 &C 的地址没有超过下界。`_save_stack` 里面还有 `assert(top - &dummy <= STACK_SIZE);` ，这个断言作用也是如此。为什么只检测下界而不检测上界？因为这里默认栈扩展方式为从高地址向低地址扩展。所以只比较栈顶 S->stack。代码里将 S->stack + STACK_SIZE 命名为 top，实际为栈底，但地址更高。

_save_stack 的 dummy 非常巧妙，其地址与栈底 top 之差就是从 dummy 到 top 之间的栈大小，栈顶就是 &dummy。假设在 dummy 之后又有声明了局部变量 x，这个局部变量并没有保存下来。因此，在 _save_stack 和 swapcontext 之间声明的任何局部变量都不应该在 swapcontext 之后被使用。显然，这里的代码符合这个规范，不会出现问题。私有栈的内存也在 _save_stack 中分配。这里采取的分配策略是按需分配，只增不减。为什么这里又不使用 realloc 而是使用 free + malloc？因为 realloc 除了重新分配内存之外，还会复制原有数据到新的内存块中。这里私有栈扩容之后，原有数据可以直接丢弃。

swapcontext 保存当前 context 到 C->ctx，再激活 S->main，也就是 main routine。

### 3.6 coroutine_status

该函数返回指定协程的状态。正如前面所说，Dead 协程已经被销毁，只是该函数返回 COROUTINE_DEAD。

### 3.7 coroutine_running

该函数返回协程组 S 正在执行的协程 id。

## 4 对协程库的扩展

### 4.1 实现嵌套 resume

如果只有一个协程组是无法实现嵌套 resume 的。但是可以创建多个协程组来达到这个目的。以下图所示的 resume 关系为例

```mermaid
flowchart
	main -->|resume| co_a
	co_a -->|yield| main
	co_a -->|resume| co_b
	co_b -->|yield| co_a
```

设计思路是 main 作为 co_a 的 resumer，co_a 作为 co_b 的 resumer，需要创建两个协程组。具体代码如下：

```c
#include "coroutine.h"
#include <stdio.h>

struct args
{
    struct schedule *next_S;
    int next_co;
};

void fa(struct schedule *S, void *ud)
{
    struct args *arg = (struct args *)ud;
    printf("fa1\n");
    coroutine_resume(arg->next_S, arg->next_co);
    printf("fa2\n");
    coroutine_resume(arg->next_S, arg->next_co);
    printf("fa3\n");
    coroutine_yield(S);
    printf("fa4\n");
}

void fb(struct schedule *S, void *ud)
{
    struct args *arg = (struct args *)ud;
    printf("fb1\n");
    coroutine_yield(S);
    printf("fb2\n");
}

int main()
{
    struct args arg1 = {0};
    struct args arg2 = {0};

    struct schedule *S_a = coroutine_open();
    struct schedule *S_b = coroutine_open();

    int co_a = coroutine_new(S_a, fa, &arg1);
    int co_b = coroutine_new(S_b, fb, &arg2);

    arg1.next_S = S_b;
    arg1.next_co = co_b;

    printf("main start\n");

    while (coroutine_status(S_a, co_a))
    {
        coroutine_resume(S_a, co_a);
        printf("main\n");
    }

    printf("main end\n");
    coroutine_close(S_a);
    coroutine_close(S_b);
}
```

这段代码输出：

```
main start
fa1
fb1
fa2
fb2
fa3
main
fa4
main
main end
```

### 4.2 实现对称协程

使用非对称协程可以实现对称协程，但是这个协程库的 yield 操作无法传递参数，只能借助全局变量。对称协程的切换流程如下图所示：
```mermaid
flowchart
	resumer -->|resume| co1
	co1 -->|yield| resumer
	resumer -->|resume| co2
	co2 -->|yield| resumer
	co1 -.->|transfer| co2
```



实现思路是协程 1 切换到协程 2 的过程可以视为协程 1 先通过 yield 返回其 resumer，再 resume 协程 2。以下是实现代码：

```cpp
#include "coroutine.h"
#include <stdio.h>

struct schedule_extra
{
	int target_co;
};

struct schedule_extra S_extra = {-1};

void co_symmetric_transfer(struct schedule *S, int id)
{
	if (coroutine_running(S) == -1)
	{
		// resumer run this branch
		coroutine_resume(S, id);
		if (S_extra.target_co != -1 && coroutine_status(S, id))
		{
			co_symmetric_transfer(S, S_extra.target_co);
		}
	}
	else
	{
		// coroutine run this branch
		S_extra.target_co = id;
		coroutine_yield(S);
	}
}

struct args
{
	int n;
	int co_other;
};

static void
foo(struct schedule *S, void *ud)
{
	struct args *arg = ud;
	int start = arg->n;
	int i;
	for (i = 0; i < 5; i++)
	{
		printf("coroutine %d : %d %d\n", coroutine_running(S), start + i, arg->co_other);
		co_symmetric_transfer(S, arg->co_other);
	}
}

static void
test(struct schedule *S)
{
	struct args arg1 = {0};
	struct args arg2 = {100};

	int co1 = coroutine_new(S, foo, &arg1);
	int co2 = coroutine_new(S, foo, &arg2);

	arg1.co_other = co2;
	arg2.co_other = co1;

	printf("main start\n");

	co_symmetric_transfer(S, co1);
	co_symmetric_transfer(S, co2);

	printf("main end\n");
}

int main()
{
	struct schedule *S = coroutine_open();
	test(S);
	coroutine_close(S);

	return 0;
}
```

这段代码输出：

```
main start
coroutine 0 : 0 1
coroutine 1 : 100 0
coroutine 0 : 1 1
coroutine 1 : 101 0
coroutine 0 : 2 1
coroutine 1 : 102 0
coroutine 0 : 3 1
coroutine 1 : 103 0
coroutine 0 : 4 1
coroutine 1 : 104 0
main end
```