---
layout: post
author: 'Wang Chen'
title: "LWN 296578: XXX"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-296578-high-but-not-too-high-res-timeouts/
description: "LWN 文章翻译，一个新的内核时间管理子系统"
category:
  - 时钟系统
  - LWN
tags:
  - Linux
  - timer
---

> 原文：[High- (but not too high-) resolution timeouts](https://lwn.net/Articles/296578/)
> 原创：By corbet @ Sept. 2, 2008
> 翻译：By [unicornx](https://github.com/unicornx) of [TinyLab.org][1]
> 校对：By [guojian-at-wowo](https://github.com/guojian-at-wowo)

> Linux provides a number of system calls that allow an application to wait for file descriptors to become ready for I/O; they include `select()`, `pselect()`, `poll()`, `ppoll()`, and `epoll_wait()`. Each of these interfaces allows the specification of a timeout putting an upper bound on how long the application will be blocked. In typical fashion, the form of that timeout varies greatly. `poll()` and `epoll_wait()` take an integer number of milliseconds; `select()` takes a `struct timeval` with microsecond resolution, and `ppoll()` and `pselect()` take a `struct timespec` with nanosecond resolution.

Linux提供了许多系统调用，允许应用程序等待文件描述符为I / O做好准备; 他们包括 选择（） ，PSELECT（） ，轮询（） ，ppoll（）和epoll_wait（） 。这些接口中的每一个都允许指定超时，从而将应用程序阻塞的时间限制在上限。以典型的方式，这种超时的形式差别很大。 poll（）和epoll_wait（）取整数毫秒; select（）采用 微秒分辨率的结构timeval，ppoll（）和pselect（） 采用结构timespec以纳秒分辨率。

> They are all the same, though, in that they convert this timeout value to jiffies, with a maximum resolution between one and ten milliseconds. A programmer might program a `pselect()` call with a 10 nanosecond timeout, but the call may not return until 10 milliseconds later, even in the absence of contention for the CPU. An error of six orders of magnitude seems like a bit much, especially given that contemporary hardware can easily support much more accurate timing.

但它们都是一样的，因为它们将此超时值转换为jiffies，其最大分辨率介于1毫秒和10毫秒之间。程序员可能会编程一个10纳秒超时的pselect（）调用，但即使在没有争用CPU的情况下，该调用也可能会在10毫秒后才会返回。六个数量级的误差似乎有点多，特别是考虑到当代硬件可以轻松支持更准确的时序。

> Arjan van de Ven recently surfaced with [a patch set](http://lwn.net/Articles/296398/) aimed at addressing this problem. The core idea is simple: have the code implementing `poll()` and `select()` use high-resolution timers instead of converting the timeout period to low-resolution jiffies. The implementation relied on a new function to provide the timeouts:

Arjan van de Ven最近出现了一个旨在解决这个问题的补丁集。核心思想很简单：让代码实现 poll（）和select（）使用高分辨率定时器，而不是将超时时间转换为低分辨率jiffies。实现依赖于一个新的函数来提供超时：

    long schedule_hrtimeout(struct timespec *time, int mode);

> Here, `time` is the timeout period, as interpreted by `mode` (which is either `HRTIMER_MODE_ABS` or `HRTIMER_MODE_REL`).

这里，时间是超时期限，由模式 （HRTIMER_MODE_ABS或HRTIMER_MODE_REL）解释。

> High-resolution timeouts are a nice feature, but one can immediately imagine a problem: higher-resolution timeouts are less likely to coincide with other events which wake up the processor. The result will be more wakeups and greater power consumption. As it happens, there are few developers who are more aware of this fact than Arjan, who has done quite a bit of work aimed at keeping processors asleep as much as possible. His solution to this problem was to only use high-resolution timeouts if the timeout period is less than one second. For longer timeout periods, the old, jiffie-based mechanism was used as before.

高分辨率超时是一个很好的功能，但可以立即想象一个问题：更高分辨率的超时不太可能与唤醒处理器的其他事件一致。结果将是更多的唤醒和更大的功耗。事实上，与Arjan相比，很少有开发人员更了解这一事实，而Arjan已经做了相当多的工作，旨在尽可能让处理器保持睡眠状态。他解决这个问题的办法是只使用高分辨率的超时时间，如果超时时间小于1秒。对于更长的超时时间，以前使用旧的基于jiffie的机制。

> Linus [didn't like that solution](https://lwn.net/Articles/296581/), calling it "ugly." His preference, instead, was to have `schedule_hrtimeout()` apply an appropriate amount of fuzz to all timeout values; the longer the timeout, the less resolution would be supplied. Alan Cox [suggested](https://lwn.net/Articles/296585/) that a better mechanism would be for the caller to supply the required accuracy with the timeout value. The problem with that idea, as Linus pointed out, is that the current system call interfaces provide no way for an application to supply the accuracy value. One could create more `poll()`-like system calls - as if there weren't enough of them already - with an accuracy parameter, but that looks like a lot of trouble to create a non-standard interface which few programmers would bother to use.

Linus 不喜欢这种解决方案，称它为“丑陋”。相反，他的选择是让schedule_hrtimeout（） 对所有超时值应用适当的模糊量; 超时时间越长，分辨率就越低。艾伦考克斯建议，更好的机制是让调用者提供超时值所需的准确性。正如Linus指出的那样，这个想法存在的问题是当前的系统调用接口不能为应用程序提供准确度值。可以创建更多的民意调查（）像系统调用 - 好像它们已经不够 - 具有精度参数，但是看起来像创建非标准接口很麻烦，很少有程序员会打扰使用它。

> A different solution came in the form of Arjan's [range-capable timer patch set](http://lwn.net/Articles/296548/). This patch extends hrtimers to accept two timeout values, called the "soft" and "hard" timeouts. The soft value - the shorter of the two - is the first time at which the timeout can expire; the kernel will make its best effort to ensure that it does not expire after the hard period has elapsed. In between the two, the kernel is free to expire the timer at any convenient time.

一种不同的解决方案以Arjan 具有范围能力的计时器补丁集的形式出现。这个补丁扩展了hrtimers以接受两个超时值，称为“软”和“硬”超时。软值 - 两者中较短的一个 - 是超时首次超时; 内核将尽最大努力确保在经过艰难时期后它不会过期。在这两者之间，内核可以随时在任何方便的时间使计时器失效。

> It's a useful feature, but it comes at the cost of some significant API changes. To begin with, the `expires` field of `struct hrtimer` goes away. Rather than manipulate `expires` directly, kernel code must now use one of the new accessor functions:

这是一个有用的功能，但它的代价是一些重要的API更改。首先，struct hrtimer的expires字段消失。内核代码现在必须使用新的访问函数之一，而不是直接操作过期：

    void hrtimer_set_expires(struct hrtimer *timer, ktime_t time);
    void hrtimer_set_expires_tv64(struct hrtimer *timer, s64 tv64);
    void hrtimer_add_expires(struct hrtimer *timer, ktime_t time);
    void hrtimer_add_expires_ns(struct hrtimer *timer, unsigned long ns);
    ktime_t hrtimer_get_expires(const struct hrtimer *timer);
    s64 hrtimer_get_expires_tv64(const struct hrtimer *timer);
    s64 hrtimer_get_expires_ns(const struct hrtimer *timer);
    ktime_t hrtimer_expires_remaining(const struct hrtimer *timer);

> Once that's done, the range capability is added to hrtimers. By default, the soft and hard expiration times are the same; code which wishes to set them independently can use the new functions:

一旦完成，范围能力就会被添加到配方。默认情况下，软和过期时间是相同的; 希望独立设置它们的代码可以使用新功能：

    void hrtimer_set_expires_range(struct hrtimer *timer, ktime_t time, 
                                   ktime_t delta);
    void hrtimer_set_expires_range_ns(struct hrtimer *timer, ktime_t time,
                                      unsigned long delta);
    ktime_t hrtimer_get_softexpires(const struct hrtimer *timer);
    s64 hrtimer_get_softexpires_tv64(const struct hrtimer *timer)

> In the new "set" functions, the specified time is the soft timeout, while `time+delta` provides the hard timeout value. There is also another form of `schedule_timeout()`:

在新的“设置”功能中，指定的时间是软超时，而时间+增量提供硬超时值。还有另一种形式的schedule_timeout（）：

    int schedule_hrtimeout_range(ktime_t *expires, unsigned long delta,
				 const enum hrtimer_mode mode);

> With this infrastructure in place, `poll()` and friends can be given approximate timeouts; the only remaining question is just how wide the range of times should be. In Arjan's patch, that range comes from two different sources. The first is a new field in the task structure called `timer_slack_ns`; as one might expect, it specifies the maximum expected timer accuracy in nanoseconds. This value can be adjusted via the `prctl()` system call. The default value is set to 50 microseconds - approximate to a certain degree, but still far more accurate than the timeouts in current kernels.

有了这个基础设施，poll（）和朋友可以得到大致的超时; 剩下的唯一问题是时间范围应该有多宽。在Arjan的补丁中，这个范围来自两个不同的来源。第一个是任务结构中称为timer_slack_ns的新字段 ; 正如人们所期望的那样，它以纳秒为单位指定最大预期定时器精度。该值可以通过prctl（）系统调用进行调整 。默认值设置为50微秒 - 接近某种程度，但仍比当前内核中的超时准确得多。

> Beyond that, though, there is a heuristic function which provides an accuracy value depending on the requested timeout period. In the case of especially long timeouts - more than ten seconds - the accuracy is set to 100ms; as the timeouts get shorter, the amount of acceptable error drops, down to a minimum of 10ns for very brief timeouts. Normally, `poll()` and company will use the value returned by the heuristic, but with the exception that the accuracy will never exceed the value found in `timer_slack_ns`.

除此之外，还有一种启发式功能，它根据请求的超时时间提供准确度值。在特别长的超时的情况下 - 超过十秒钟 - 准确度被设置为100ms; 随着超时时间的缩短，可接受的错误数量将会下降，对于非常短暂的超时，可以降至最小10ns。通常， poll（）和company将使用启发式返回的值，但不同之处在于精度不会超过timer_slack_ns中的值。

> The end result is the provision of more accurate timeouts on the polling functions while, simultaneously, preserving the ability to combine timeouts with other system events.

最终结果是在轮询功能上提供更准确的超时，同时保留将超时与其他系统事件组合的能力。

[1]: http://tinylab.org
