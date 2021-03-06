> 不要通过共享内存来通信，而应该通过通信来共享内存。这是作为 Go 语言的主要创造者之一的 Rob Pike 的至理名言，这也充分体现了 Go 语言最重要的编程理念。

#### 内核态线程与用户态线程
说起并发模型，就一定要先聊线程，因为无论何种并发模型，到了操作系统层面，一定是以线程的形态发生的。我们知道，线程是处理器调度的基本单元，进程是资源分配的基本单位。每个进程启动时会由系统自动创建一个线程去执行任务，叫主线程。 单个进程中也可以存在多个线程，这些线程都是由已存在的线程创建出来的，创建的方式是通过系统调用。

根据资源访问权限的不同, 进程的运行空间分为 内核空间 和 用户空间。
- 内核空间：具有最高权限，可以直接访问所有资源，进程在内核空间内运行时，称为内核态
- 用户空间：访问受限的资源，不能直接访问内存等硬件设备，必须通过系统调用到内核中，才能访问特权资源，进程在用户空间时，称为用户态

进程是由内核来管理和调度的，进程的切换只能发生在内核态，所以进程上下文不仅包括了虚拟内存、栈、全局变量等用户空间资源，还包括了内核堆栈、寄存器等内核空间的状态。

从用户态到内核态的转变，需要通过系统调用来完成，具体来说：
- 一次系统调用的过程，会发生两次CPU上下文切换
- 系统调用过程中一直是同一个进程在运行
- 系统调用过程通常称为特权模式切换
- 先将CPU寄存器里用户态的指令位置，保存起来
- CPU寄存器加载内核态指令位置
- 跳转到内核态运行任务
- 系统调用结束后，CPU寄存器恢复原来保存的用户态，再切换到用户空间，继续运行进程

我们前面已经说过了，线程是调度的基本单位，而进程是资源拥有的基本单位。那么该怎么理解这句话呢？事实上，内核中的任务调度，调度的对象是线程，而进程只是给线程提供了虚拟内存、全局变量等资源。

- 当进程只有一个线程时，可以认为进程就等于线程
- 当进程拥有多个线程时，线程间会共享虚拟内存，这些资源在上下文切换时不需要修改
- 线程也有自己的私有数据，比如栈和寄存器，这些在上下文切换时需要保留

#### 线程实现模型
线程的实现模型主要有3类，分别是内核级线程模型、用户级线程模型和两级线程模型。
- 内核级线程模型
  - 用户线程与内核调度实体(内核线程)是一对一的关系，不同线程之间互不影响
  - 实现简单，直接借助OS提供的线程能力，可以跑在多个CPU核心上，实现了真正的并行
  - 每个线程由内核调度器独立调度，如果一个线程阻塞则不影响其他的线程
  - 但其创建、销毁以及上下文切换都是直接由内核来处理，性能影响大
  - 大部分编程语言的线程库(Linux的pthread, Java的java.lang.Thread, C++的std::thread等)都是直接对操作系统的线程进行一层封装
- 用户级线程模型（协程）
  - 用户线程与内核调度实体(内核线程)是多对一的关系，多个用户线程映射到一个内核线程上
  - 协程就是基于这种设计实现的
  - 协程的创建、销毁以及多个线程之间的协调等操作都是由用户自己实现的协程库来负责的
  - 这种方式更轻量化，对系统资源的消耗更小，可以创建的数量更多，上下文切换所花费的代价更低
  - 但如果某个协程上调用了阻塞式的系统调用(如用阻塞方式read网络IO)，那么基于此内核线程的上的所有协程都会被阻塞
  - 所以现在的协程库都会把自己的阻塞操作封装为非阻塞的形式，主动让出自己，让线程去执行其他协程，然后通过某种通知机制来让线程回来
- 混合线程模型(两级线程模型)
  - 用户线程与内核调度实体(内核线程)是多对多(M:N)的关系，其综合了上面两种模型的优点
  - 当某个内核线程因其上的协程任务阻塞而被内核调度出CPU时，当前与其关联的其他协程可以重新与其他内核线程建立关联
  - 这种动态关联机制的实现很复杂，需要用户自己实现，没错，Go 就是用的这种模型
  - Go 的 goroutine 调度器实现了协程到内核调度实体的调度，内核调度器实现了内核调度实体到CPU上的调度

#### Goroutine 与 线程的区别
- 内存占用
  - 创建一个 goroutine 的栈内存消耗约为 2KB, 在运行过程中, 如果栈空间不足, 会自动扩容.
  - 为了避免线程栈溢出, os 会为线程分配一个较大的栈内存(1-8MB), 而且还需要 guard page 来与其他线程栈空间进行隔离, 栈空间一旦初始化大小就不会再变.
- 资源消耗
  - 创建/销毁线程是内核管理的, 需要陷入内核, 造成更大的系统开销
  - goroutine 是用户态线程, 是由 go runtime 管理, 创建和销毁的消耗非常小
- 调度切换
  - 抛开陷入内核，线程切换会消耗 1000-1500 纳秒(上下文保存成本高，较多寄存器，公平性，复杂时间计算统计)，一个纳秒平均可以执行 12-18 条指令。
  - 所以由于线程切换，执行指令的条数会减少 12000-18000。goroutine的切换约为 200 ns(用户态、3个寄存器)，相当于 2400-3600 条指令。因此，goroutines切换成本比 threads 要小得多。
- 复杂性
  - 线程的创建和退出复杂，多个thread间通讯复杂(share memory)。
  - 不能大量创建线程(参考早期的httpd)，成本高，使用网络多路复用，存在大量callback(参考twemproxy、nginx的代码)。对于应用服务线程门槛高，例如需要做第三方库隔离，需要考虑引入线程池等。

#### Go 并发调度模型：G-P-M模型
在操作系统提供的内核线程之上，Go搭建了一个特有的两级线程模型。goroutine机制实现了M:N的线程模型，goroutine机制是协程的一种实现。
Go语言内置的调度器负责 goroutine 协程与内核线程之间的调度，所以理解 goroutine 机制的关键是理解这个调度器(scheduler)的实现

Go 1.2前的调度器实现，限制了 Go 并发程序的伸缩性，尤其是对那些有高吞吐或并行计算需求的服务程序。

每个 goroutine 对应于 runtime 中的一个抽象结构：G，而 thread 作为“物理 CPU”的存在而被抽象为一个结构：M(machine)。当 goroutine 调用了一个阻塞的系统调用，运行这个 goroutine 的线程就会被阻塞，这时至少应该再创建一个线程来运行别的没有阻塞的 goroutine。线程这里可以创建不止一个，可以按需不断地创建，而活跃的线程（处于非阻塞状态的线程）的最大个数存储在变量 GOMAXPROCS中。

GM 调度模型的问题:
  - 单一全局互斥锁(Sched.Lock)和集中状态存储
    - 导致所有 goroutine 相关操作，比如：创建、结束、重新调度等都要上锁。
  - Goroutine 传递问题
    - M 经常在 M 之间传递”可运行”的 goroutine，这导致调度延迟增大以及额外的性能损耗（刚创建的 G 放到了全局队列，而不是本地 M 执行，不必要的开销和延迟）。
  - Per-M 持有内存缓存 (M.mcache)
    - 每个 M 持有 mcache 和 stack alloc，然而只有在 M 运行 Go 代码时才需要使用的内存(每个 mcache 可以高达2mb)，当 M 在处于 syscall 时并不需要。运行 Go 代码和阻塞在 syscall 的 M 的比例高达1:100，造成了很大的浪费。同时内存亲缘性也较差。G 当前在 M运 行后对 M 的内存进行了预热，因为现在 G 调度到同一个 M 的概率不高，数据局部性不好。
  - 严重的线程阻塞/解锁
    - 在系统调用的情况下，工作线程经常被阻塞和取消阻塞，这增加了很多开销。比如 M 找不到G，此时 M 就会进入频繁阻塞/唤醒来进行检查的逻辑，以便及时发现新的 G 来执行。

Dmitry Vyukov的方案是引入一个结构 P，它代表了 M 所需的上下文环境，也是处理用户级代码逻辑的处理器。它负责衔接 M 和 G 的调度上下文，将等待执行的 G 与 M 对接。当 P 有任务时需要创建或者唤醒一个 M 来执行它队列里的任务。所以 P/M 需要进行绑定，构成一个执行单元。P 决定了并行任务的数量，可通过 runtime.GOMAXPROCS 来设定。在 Go1.5 之后GOMAXPROCS 被默认设置可用的核数，而之前则默认为1。

Go语言调度器(scheduler)由4个主要结构组成，分别是M、G、P、Sched，前三个定义在 runtime.h 中，Sched 定义在 proc.c 中
- M结构是 Machine，系统线程，也就是我们说的内核线程，它由操作系统管理，goroutine 跑在 Machine 之上。
- P结构是 Processor，处理器，主要用途是用来执行 goroutine，维护了一个 goroutine 队列，即 runqueue
- G结构是 goroutine 实现的核心结构，包含了栈、指令指针以及其他对调度 goroutine 很重要的信息
- Sched结构是调度器，维护 Machine 和 goroutine 队列以及调度器的一些状态信息等

![GMP](./assets/gmp.png ':size=400')

引入了 local queue，因为 P 的存在，runtime 并不需要做一个集中式的 goroutine 调度，每一个 M 都会在 P's local queue、global queue 或者其他 P 队列中找 G 执行，减少全局锁对性能的影响。
这也是 GMP Work-stealing 调度算法的核心。注意 P 的本地 G 队列还是可能面临一个并发访问的场景，为了避免加锁，这里 P 的本地队列是一个 LockFree的队列，窃取 G 时使用 CAS 原子操作来完成。关于LockFree 和 CAS 的知识参见 Lock-Free。

#### Work-stealing 调度算法

当一个 P 执行完本地所有的 G 之后，会尝试挑选一个受害者 P，从它的 G 队列中窃取一半的 G。当尝试若干次窃取都失败之后，会从全局队列中获取(当前个数/GOMAXPROCS)个 G。

为了保证公平性，从随机位置上的 P 开始，而且遍历的顺序也随机化了(选择一个小于 GOMAXPROCS，且和它互为质数的步长)，保证遍历的顺序也随机化了。

光窃取失败时获取是不够的，可能会导致全局队列饥饿。P 的调度算法中还会每个 N 轮调度之后就去全局队列拿一个 G。

谁放入的全局队列呢？
新建 G 时 P 的本地 G 队列放不下已满并达到256个的时候会放半数 G 到全局队列去，阻塞的系统调用返回时找不到空闲 P 也会放到全局队列。

#### Syscall

调用 syscall 后会解绑 P，然后 M 和 G 进入阻塞，而 P 此时的状态就是 syscall，表明这个 P 的 G 正在 syscall 中，这时的 P 是不能被调度给别的 M 的。如果在短时间内阻塞的 M 就唤醒了，那么 M 会优先来重新获取这个 P，能获取到就继续绑回去，这样有利于数据的局部性。

系统监视器 (system monitor)，称为 sysmon，会定时扫描。在执行 syscall 时, 如果某个 P 的 G 执行超过一个 sysmon tick(10ms)，就会把他设为 idle，重新调度给需要的 M，强制解绑。

P1 和 M 脱离后目前在 idle list 中等待被绑定（处于 syscall 状态）。而 syscall 结束后 M 按照如下规则执行直到满足其中一个条件：
  - 尝试获取同一个 P(P1)，恢复执行 G
  - 尝试获取 idle list 中的其他空闲 P，恢复执行 G
  - 找不到空闲 P，把 G 放回 global queue，M 放回到 idle list

当使用了 Syscall，Go 无法限制 Blocked OS threads 的数量, 使用 syscall 写程序要认真考虑 pthread exhaust 问题。

#### Spining thread

线程自旋是相对于线程阻塞而言的，表象就是循环执行一个指定逻辑(调度逻辑，目的是不停地寻找 G)。这样做的问题显而易见，如果 G 迟迟不来，CPU 会白白浪费在这无意义的计算上。但好处也很明显，降低了 M 的上下文切换成本，提高了性能。在两个地方引入自旋：
  - 类型1：M 不带 P 的找 P 挂载（一有 P 释放就结合）
  - 类型2：M 带 P 的找 G 运行（一有 runable 的 G 就执行）

为了避免过多浪费 CPU 资源，自旋的 M 最多只允许 GOMAXPROCS (Busy P)。同时当有类型1的自旋 M 存在时，类型2的自旋 M 就不阻塞，阻塞会释放 P，一释放 P 就马上被类型1的自旋 M 抢走了，没必要。

在新 G 被创建、M 进入系统调用、M 从空闲被激活这三种状态变化前，调度器会确保至少有一个自旋 M 存在（唤醒或者创建一个 M），除非没有空闲的 P。
  - 当新 G 创建，如果有可用 P，就意味着新 G 可以被立即执行，即便不在同一个 P 也无妨，所以我们保留一个自旋的 M（这时应该不存在类型1的自旋只有类型2的自旋）就可以保证新 G 很快被运行。
  - 当 M 进入系统调用，意味着 M 不知道何时可以醒来，那么 M 对应的 P 中剩下的 G 就得有新的 M 来执行，所以我们保留一个自旋的 M 来执行剩下的 G（这时应该不存在类型2的自旋只有类型1的自旋）。
  - 如果 M 从空闲变成活跃，意味着可能一个处于自旋状态的 M 进入工作状态了，这时要检查并确保还有一个自旋 M 存在，以防还有 G 或者还有 P 空着的。

#### 总结 GMP 模型
- 单一全局互斥锁(Sched.Lock)和集中状态存储
  - G 被分成全局队列和 P 的本地队列，全局队列依旧是全局锁，但是使用场景明显很少，P 本地队列使用无锁队列，使用原子操作来面对可能的并发场景。
- Goroutine 传递问题
  - G 创建时就在 P 的本地队列，可以避免在 G 之间传递（窃取除外），G 对 P 的数据局部性好; 当 G 开始执行了，系统调用返回后 M 会尝试获取可用 P，获取到了的话可以避免在 M 之间传递。而且优先获取调用阻塞前的 P，所以 G 对 M 数据局部性好，G 对 P 的数据局部性也好。
- Per-M 持有内存缓存 (M.mcache)
  - 内存 mcache 只存在 P 结构中，P 最多只有 GOMAXPROCS 个，远小于 M 的个数，所以内存没有过多的消耗。
- 严重的线程阻塞/解锁
  - 通过引入自旋，保证任何时候都有处于等待状态的自旋 M，避免在等待可用的 P 和 G 时频繁的阻塞和唤醒。

#### Channel 通道
虽然我们在 Go 语言中也能使用共享内存加互斥锁进行通信，但是 Go 语言提供了一种不同的并发模型，也就是通信顺序进程（Communicating sequential processes，CSP）1。Goroutine 和 Channel 分别对应 CSP 中的实体和传递信息的媒介，Go 语言中的 Goroutine 会通过 Channel 传递数据。

在Go语言中，要传递某个数据给另一个 goroutine(协程)，可以把这个数据封装成一个对象，然后把这个对象的指针传入某个channel中，另一个goroutine从这个channel中读出这个指针，并处理其指向的内存对象。Go从语言层面保证同一时间只有一个goroutine能够访问channel里面的数据，为开发者提供了一种优雅简单的工具，所以Go的做法就是使用channel来通信，通过通信来传递内存数据，使得内存数据在不同的goroutine中传递，而不是使用共享内存来通信
