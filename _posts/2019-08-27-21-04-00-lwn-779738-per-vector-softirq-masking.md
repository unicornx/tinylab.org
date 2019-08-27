---
layout: post
draft: true
author: 'Wang Chen'
title: "LWN 779738: xxxx"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-779738/
description: "LWN 中文翻译，xxxx"
category:
  - 进程调度
  - LWN
tags:
  - Linux
  - realtime
  - softirq
---

**请点击 [LWN 中文翻译计划](/lwn)，了解更多详情。**

> 原文：[Per-vector software-interrupt masking](https://lwn.net/Articles/779738/)
> 原创：By Jonathan Corbet @ Feb. 15, 2019
> 翻译：By [unicornx](https://github.com/unicornx)
> 校对：By [xxx](https://github.com/xxx)

> Software interrupts (or "softirqs") are one of the oldest deferred-execution mechanisms in the kernel, and that age shows at times. Some developers have been occasionally heard to mutter about removing them, but softirqs are too deeply embedded into how the kernel works to be easily ripped out; most developers just leave them alone. So the recent [per-vector softirq masking patch set](https://lwn.net/ml/linux-kernel/20190212171423.8308-1-frederic@kernel.org/) from Frederic Weisbecker is noteworthy as an exception to that rule. Weisbecker is not getting rid of softirqs, but he is trying to reduce their impact and improve their latency.

软件中断（或“softirqs”）是内核中最古老的延迟执行机制之一，有时会显示该年龄。偶尔会听到一些开发人员嘀咕着要删除它们，但是softirqs太深入地嵌入了内核如何轻易地被删除; 大多数开发人员只是让他们孤单 因此，Frederic Weisbecker 最近的每个矢量softirq掩蔽补丁集值得注意，作为该规则的一个例外。Weisbecker没有摆脱softirqs，但他正试图减少他们的影响并改善他们的延迟。

> Hardware interrupts are the means by which the hardware can gain a CPU's attention to signal the completion of an I/O operation or some other situation of interest. When an interrupt is raised, the currently running code is (usually) preempted and an interrupt handler within the kernel is executed. A cardinal rule for interrupt handlers is that they must execute quickly, since they interfere with the other work the CPU is meant to be doing. That usually implies that an interrupt handler will do little more than acknowledge the interrupt to the hardware and set aside enough information to allow the real processing work to be done in a lower-priority mode.

硬件中断是硬件可以获得CPU注意以指示I / O操作完成或其他一些感兴趣的情况的手段。当引发中断时，（通常）抢占当前运行的代码并执行内核中的中断处理程序。中断处理程序的基本规则是它们必须快速执行，因为它们会干扰CPU本来要做的其他工作。这通常意味着中断处理程序只会确认硬件中断，并留出足够的信息以允许在较低优先级模式下完成实际处理工作。

> The kernel offers a number of deferred-execution mechanisms through which that work can eventually be done. In current kernels, the most commonly used of those is workqueues, which can be used to queue a function call to be run in kernel-thread context at some later time. Another is tasklets, which execute at a higher priority than workqueues; adding new tasklet users tends to be mildly discouraged for reasons we'll get to. Other kernel subsystems might use timers or dedicated kernel threads to get their deferred work done.

内核提供了许多延迟执行机制，通过这些机制最终可以完成这项工作。在当前的内核中，最常用的是工作队列，它可以用来对函数调用进行排队，以便稍后在内核线程上下文中运行。另一个是tasklet，它的执行优先级高于工作队列; 由于我们要达到的原因，添加新的tasklet用户往往会被温和地劝阻。其他内核子系统可能使用计时器或专用内核线程来完成延迟工作。

## 软中断（Softirqs）

> Then, there are softirqs which, as their name would suggest, are a software construct; they are patterned after hardware interrupts, but hardware interrupts are enabled while software interrupts execute. Softirqs have assigned numbers ("vectors"); "raising" a particular softirq will cause the handler function for the indicated vector to be called at a convenient time in the near future. That "convenient time" is usually either at the end of hardware-interrupt processing or when a processor that has disabled softirq processing re-enables it. Softirqs thus run outside of the CPU scheduler as a relatively high-priority activity.

然后，有一些软件，正如他们的名字所暗示的那样，是软件结构; 它们在硬件中断后被图案化，但硬件中断在软件中断执行时被启用。Softirqs已经分配了数字（“向量”）; “提升”特定的softirq将导致在不久的将来在方便的时间调用指示的向量的处理函数。“方便时间”通常是在硬件中断处理结束时或者已禁用softirq处理的处理器重新启用它。因此，Softirqs作为相对高优先级的活动在CPU调度程序之外运行。

> In the 5.0-rc kernel, there are ten softirq vectors defined:

在5.0-rc内核中，定义了10个softirq向量：

> - `HI_SOFTIRQ` and `TASKLET_SOFTIRQ` are both for the execution of tasklets; this is part of why tasklets are discouraged. High-priority tasklets are run ahead of any other softirqs, while normal-priority tasklets are run in the middle of the pack.
> - `TIMER_SOFTIRQ` is for the handling of timer events. `HRTIMER_SOFTIRQ` is also defined; it was once used for high-resolution timers, but that has not been the case since [this change](https://git.kernel.org/linus/c6eb3f70d448) was made for the 4.2 release.
> - `NET_TX_SOFTIRQ` and `NET_RX_SOFTIRQ` are used for network transmit and receive processing, respectively.
> - `BLOCK_SOFTIRQ` handles block I/O completion events; this functionality was [moved to softirq](https://git.kernel.org/linus/ff856bad67cb) mode for the 2.6.16 kernel in 2006.
> - `IRQ_POLL_SOFTIRQ` is used by the [irq_poll mechanism](https://elixir.bootlin.com/linux/v5.0-rc6/source/lib/irq_poll.c), which was [generalized](https://git.kernel.org/linus/511cbce2ff) from the block interrupt-polling mechanism for the 4.5 release in 2015. Its predecessor, `BLOCK_IOPOLL_SOFTIRQ` was [added](https://git.kernel.org/linus/5e605b64a183) for the 2.6.32 release in 2009; no softirq vectors have been added since then.
> - `SCHED_SOFTIRQ` is used by the scheduler to perform load-balancing and other scheduling tasks.
> - `RCU_SOFTIRQ` performs read-copy-update processing. There was [an attempt](https://git.kernel.org/linus/a26ac2455ffcf3) made by the late Shaohua Li in 2011 to move this processing to a kernel thread, but performance regressions forced that change to be reverted shortly thereafter.

- HI_SOFTIRQ 和 TASKLET_SOFTIRQ 都用于执行tasklet ; 这是为什么不鼓励使用tasklet的一部分。高优先级的tasklet在任何其他softirq之前运行，而普通优先级的tasklet在包的中间运行。
- TIMER_SOFTIRQ 用于处理计时器事件。 HRTIMER_SOFTIRQ也已定义; 它曾经被用于高分辨率的计时器，但事实并非如此，因为这一变化是针对4.2发布的。
- NET_TX_SOFTIRQ 和 NET_RX_SOFTIRQ 分别用于网络发送和接收处理。
- BLOCK_SOFTIRQ处理块I / O完成事件; 这个功能 在2006年被转移到 2.6.16内核的softirq模式。
- IRQ_POLL_SOFTIRQ 由irq_poll机制使用，该机制是从2015年4.5版本的块中断轮询机制推广而来的。它的前身BLOCK_IOPOLL_SOFTIRQ是在2009年的2.6.32版本中添加的。从那时起，没有添加softirq载体。
- 调度程序使用 SCHED_SOFTIRQ 来执行负载平衡和其他调度任务。
- RCU_SOFTIRQ 执行读取 - 复制 - 更新处理。有 一个尝试 该处理移动到内核线程由已故的李韶华在2011年提出，但性能衰退迫使这一变化此后不久被恢复。

Thomas Gleixner once [summarized](https://lwn.net/Articles/518993/) the software-interrupt mechanism as "`a conglomerate of mostly unrelated jobs, which run in the context of a randomly chosen victim w/o the ability to put any control on them`". For historical reasons that long predate Linux, software interrupts also sometimes go by the name "bottom halves" — they are the half of interrupt processing that is done outside of hardware interrupt mode. For this reason, one will often see the term "BH" used to refer to software interrupts.

托马斯·格莱克纳（Thomas Gleixner）曾将软件中断机制概括为“ 一个大多数不相关的工作集团，这些工作在一个随机选择的受害者的背景下运行，无法控制他们 ”。由于早于Linux的历史原因，软件中断有时也会被称为“下半部分” - 它们是在硬件中断模式之外完成的中断处理的一半。出于这个原因，人们经常会看到术语“BH”用于指代软件中断。

> Since software interrupts execute at a high priority, they can create high levels of latency in the system if they are not carefully managed. As little work as possible is done in softirq mode, but certain kinds of system workloads (high network traffic, for example) can still cause softirq processing to adversely impact the system as a whole. The kernel will actually kick softirq handling out to a set of `ksoftirqd` kernel threads if it starts taking too much time, but there can be performance costs even if the total CPU time used by softirq processing is relatively low.

由于软件中断以高优先级执行，因此如果不仔细管理，它们可能会在系统中产生高级别的延迟。尽管在softirq模式下尽可能少地完成工作，但某些类型的系统工作负载（例如，高网络流量）仍然可能导致softirq处理对整个系统产生负面影响。如果内核开始花费太多时间，内核实际上会将softirq处理到一组ksoftirqd内核线程，但即使softirq处理使用的总CPU时间相对较低，也可能会产生性能成本。

## Softirq并发 Softirq concurrency

> Part of the problem, especially for latency-sensitive workloads, results from the fact that softirqs are another source of concurrency in the system that must be controlled. Any work that might try to access data concurrently with a softirq handler must use some sort of mutual exclusion mechanism and, since softirqs are essentially interrupts, special care must be taken to avoid deadlocks. If, for example, a kernel function acquires a spinlock, but is then interrupted by a softirq that tries to take the same lock, that softirq handler will wait forever — the sort of situation that latency-sensitive users tend to get especially irritable over.

问题的一部分，特别是对于延迟敏感的工作负载，是因为softirqs是必须控制的系统中的另一个并发源。任何可能尝试与softirq处理程序同时访问数据的工作都必须使用某种互斥机制，因为softirqs本质上是中断，所以必须特别注意避免死锁。例如，如果内核函数获取了一个自旋锁，但是后来被尝试采用相同锁的softirq中断，那么softirq处理程序将永远等待 - 这种对延迟敏感的用户往往特别烦躁的情况。

> To avoid such problems, the kernel provides a number of ways to prevent softirq handlers from running for a period of time. For example, a call to `spin_lock_bh()` will acquire the indicated spinlock and also disable softirq processing for as long as the lock is held, preventing the deadlock scenario described above. Any subsystem that uses software interrupts must take care to ensure that they are disabled in places where unwanted concurrency could occur.

为避免此类问题，内核提供了许多方法来防止softirq处理程序运行一段时间。例如，对spin_lock_bh（）的调用 将获取指示的自旋锁，并且只要锁定被保持就禁用softirq处理，从而防止上述死锁情况。任何使用软件中断的子系统都必须注意确保在可能发生意外并发的地方禁用它们。

> Linux software interrupts have an interesting problem — interesting because it is seemingly obvious but has been there since the beginning. The softirq vectors described above are all independent of each other, and their handlers are unlikely to interfere with each other. Network transmit processing should not be bothered if the block softirq handler runs concurrently, for example. So code that must protect against concurrent access from a softirq handler need only disable the one handler that it might race with, but functions like `spin_lock_bh()` disable ***all*** softirq handling. That can cause unrelated handlers to be delayed needlessly, once again leading to bad temper in the low-latency camp.

Linux软件中断有一个有趣的问题 - 有趣的是因为它看起来很明显，但从一开始就存在。上述softirq向量都是彼此独立的，并且它们的处理程序不太可能相互干扰。例如，如果块softirq处理程序同时运行，则不应该打扰网络传输处理。因此，必须防止来自softirq处理程序的并发访问的代码只需要禁用它可能会竞争的一个处理程序，但像spin_lock_bh（）这样的函数会禁用所有 softirq处理。这可能导致不相关的处理程序不必要地延迟，再次导致低延迟阵营中的坏脾气。

## Per-vector masking

> Weisbecker's answer to this is to allow individual softirq vectors to be disabled while the others remain enabled. The [first attempt](https://lwn.net/ml/linux-kernel/1539213137-13953-1-git-send-email-frederic@kernel.org/), posted in October 2018, changed the prototypes of functions like `spin_lock_bh()`, `local_bh_disable()`, and `rcu_read_lock_bh()` to contain a mask of the vectors to disable. There was just one little problem: there are a lot of callers to those functions in the kernel. So the bottom line for that patch set was:

Weisbecker对此的回答是允许禁用单个softirq向量，而其他向量保持启用状态。2018年10月发布的第一次尝试改变了spin_lock_bh（），local_bh_disable（）和 rcu_read_lock_bh（）等函数的原型，以包含要禁用的向量的掩码。只有一个小问题：内核中有很多这些函数的调用者。所以补丁集的底线是：

```
    945 files changed, 13857 insertions(+), 9767 deletions(-)
```

> The kernel community has gotten good at merging large, invasive patch sets, but that one still pushed the limits a bit. That is especially true given that almost all call sites still disabled all vectors; doing anything else requires careful auditing of every change. The second time around, Weisbecker decided to take an easier approach and define new functions, leaving the old ones unchanged. So this patch set introduces functions like:

内核社区已经擅长合并大型的侵入式补丁集，但是那个仍然有点推动了限制。鉴于几乎所有呼叫站点仍然禁用所有向量，这一点尤其如此; 做其他事情需要仔细审核每一个变化。第二次，Weisbecker决定采用更简单的方法并定义新的功能，保持原有功能不变。所以这个补丁集引入了如下函数：

```
    unsigned int spin_lock_bh_mask(spinlock_t *lock, unsigned int mask);
    unsigned int local_bh_disable_mask(unsigned int mask);
    /* ... */
```

> After the call, only the softirq vectors indicated by the given `mask` will have been disabled; the rest can still be run if they were enabled before the call. The return value of these functions is the previous set of masked softirqs; it is needed when renabling softirqs to their previous state.

通话结束后，只有给定掩码指示的softirq向量 才会被禁用; 如果在通话之前启用了其余部分，则仍然可以运行其余部分。这些函数的返回值是前一组被屏蔽的softirqs; 将softirqs重置为先前状态时需要它。

> This patch set is rather less intrusive:

这个补丁集的侵入性较小：

```
    36 files changed, 690 insertions(+), 281 deletions(-)
```

> That is true even though it goes beyond the core changes to, for example, add support to the lockdep locking checker to ensure that the use of the vector masks is consistent. One thing that has not yet been done is to allow one softirq handler to preempt another; that's on the list for future work.

即使超出核心更改，也是如此，例如，添加对lockdep锁定检查器的支持，以确保矢量掩码的使用是一致的。尚未完成的一件事是允许一个softirq处理程序抢占另一个; 这是未来工作的清单。

> No performance numbers have been provided, so it is not possible to know for sure that this work has achieved its goal of providing better latencies for specific softirq handlers. Still, networking maintainer David Miller [indicated his approval](https://lwn.net/ml/linux-kernel/20190212.102912.1498886718723769885.davem@davemloft.net/), saying: "`I really like this stuff, nice work`". Linus Torvalds had some low-level comments that will need to be addressed in the next iteration of the patch set. Some other important reviewers have yet to weigh in, so it would be too soon to say that this work is nearly ready. But, in the absence of a complete removal of softirqs, there is clear value in not disabling them needlessly, so this change seems likely to vector itself into the mainline sooner or later.

没有提供性能数据，因此无法确定此项工作是否已达到为特定softirq处理程序提供更好延迟的目标。不过，网络维护者David Miller 表示赞同，他说：“ 我真的很喜欢这些东西，很棒的工作 ”。Linus Torvalds有一些低级别的评论需要在补丁集的下一次迭代中解决。其他一些重要的评论者还没有权衡，所以现在说这项工作已接近准备还为时尚早。但是，在没有完全删除softirqs的情况下，没有不必要地禁用它们有明显的价值，因此这种变化似乎迟早会将自身转化为主线。

**请点击 [LWN 中文翻译计划](/lwn)，了解更多详情。**

[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0a0337e0d1d134465778a16f5cbea95086e8e9e0
[2]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aac453635549699c13a84ea1456d5b0e574ef855
[3]: https://kernelnewbies.org/Linux_4.6#Improve_the_reliability_of_the_Out_Of_Memory_task_killer
[4]: https://lwn.net/Articles/668133/
[5]: https://lwn.net/Articles/627419/
[6]: https://lwn.net/Articles/666024/
