---
layout: post
draft: true
author: 'Wang Chen'
title: "LWN 735887: xxx"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-735887/
description: "LWN 文章翻译，xxx"
category:
  - 时钟系统
  - LWN
tags:
  - Linux
  - timer
---

**请点击 [LWN 中文翻译计划](/lwn)，了解更多详情。**

> 原文：[Improving the kernel timers API](https://lwn.net/Articles/735887/)
> 原创：By Jonathan Corbet @ Oct. 9, 2017
> 翻译：By [unicornx](https://github.com/unicornx)
> 校对：By [xxx](https://github.com/xxx)

> The kernel's timer interface has been around for a long time, and its API shows it. Beyond a lack of conformance with current in-kernel interface patterns, the timer API is not as efficient as it could be and stands in the way of ongoing kernel-hardening efforts. A late addition to the 4.14 kernel paves the way toward a wholesale change of this API to address these problems.

内核的计时器接口已存在很长时间了，它的API显示它。除了缺乏与当前内核中接口模式的一致性之外，定时器API并不像它可能那样高效，并阻碍了正在进行的内核强化工作。4.14内核的后期添加为批量更改此API以解决这些问题铺平了道路。

> It is worth noting that the kernel has two core timer mechanisms. One of those — the high-resolution timer (or "hrtimer") — subsystem, is focused on near-term events where the timer is expected to run to completion. The other subsystem is just called "kernel timers"; it offers less precision but is more efficient in situations where the timer will probably be canceled before it fires. There are many places in the kernel where timers are used to detect when a device or a network peer has failed to respond within the expected time; when, as usual, the expected response does happen, the timer is canceled. Kernel timers are well suited to that kind of use. The work at hand focuses on that second type of timer.

值得注意的是，内核有两个核心计时器机制。其中之一 - 高分辨率计时器（或“hrtimer”） - 子系​​统，专注于预计计时器运行完成的近期事件。另一个子系统叫做“内核定时器”; 它提供的精度较低，但在计时器可能会在发生之前被取消的情况下效率更高。内核中有许多地方使用定时器来检测设备或网络对等体何时无法在预期时间内响应; 当像往常一样，预期的响应确实发生时，计时器被取消。内核定时器非常适合这种用途。手头的工作集中在第二种计时器上。

> Kernel timers are described by the `timer_list` structure, defined in [`<linux/timer.h>`](http://elixir.free-electrons.com/linux/v4.14-rc4/source/include/linux/timer.h):

内核定时器由timer_list结构描述，在<linux / timer.h>中定义：

    struct timer_list {
	unsigned long		expires;
	void			(*function)(unsigned long);
	unsigned long		data;
	/* ... other stuff elided ... */
    }

> The `expires` field contains the expiration time of the timer (in jiffies); on expiration, `function()` will be called with the given `data` value. It is possible to fill in a `timer_list` structure manually, but it is more common to use the `setup_timer()` macro:

的期满字段包含了定时器（以jiffies）的到期时间; 在到期时，将使用给定的数据值调用 function（）。可以 手动填写timer_list结构，但更常见的是使用setup_timer（） 宏：

    void setup_timer(timer, function, data);

> There are a number of issues with this API, as [argued by Kees Cook](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=686fef928bba6be13cabe639f154af7d72b63120). The `data` field bloats the `timer_list` structure unnecessarily and, as an unadorned `unsigned long` value, it resists any sort of type checking. It is not uncommon for callers to cast pointer values to and from this value, for example. For these reasons, it is far more common in current kernel APIs to dispense with the `data` field and, instead, just pass a pointer to the relevant structure (the `timer_list` structure in this case) to the callback. Likely as not, that structure is embedded within a larger structure containing the information the callback needs anyway, so a simple `container_of()` call can replace the casting of the `unsigned long` value.

正如Kees Cook所说，这个API存在许多问题。该数据字段的涨大 timer_list结构不必要的和，作为缦 无符号长值，它抵抗任何种类的类型检查。例如，调用者将指针值转换为此值或从该值转换指针值的情况并不少见。由于这些原因，在当前的内核API中放弃数据字段更为常见，而只是将指针传递给相关结构（在本例中为timer_list结构）到回调。可能没有，该结构嵌入在一个更大的结构中，包含回调所需的信息，所以一个简单的 container_of（）call可以替换unsigned long值的转换。

> As might be expected, though, Cook has concerns about this API that go beyond matching the current kernel style. One of those is that a buffer overflow in the area of a `timer_list` structure may be able to overwrite both the function pointer and the data passed to the called function, allowing arbitrary calls within the kernel. That, naturally, makes `timer_list` structures interesting to attackers, and explains why Cook has been [trying to harden timers](https://lwn.net/Articles/731082/) for a while. The prototype of the timer callback, containing a single `unsigned long` argument, is also evidently an impediment to "future control flow integrity work". It would be better if the callback had a unique prototype that was visibly different from all of the other kernel functions taking an `unsigned long` argument.

然而，正如可以预料的那样，Cook对此API的关注超出了与当前内核样式的匹配。其中之一是timer_list结构区域中的缓冲区溢出可能会覆盖函数指针和传递给被调用函数的数据，从而允许在内核中进行任意调用。这自然使得timer_list结构对攻击者来说很有趣，并解释了为什么库克一直试图强化计时器 的原因。定时器回调的原型，包含一个无符号长 参数，显然也是“ 未来控制流完整性工作的障碍”如果回调有一个独特的原型，与采用无符号长参数的所有其他内核函数明显不同，那将会更好。

> Cook has been working on changes to the timer interface for a while in an attempt to address these issues. The core idea is simple: get rid of the `data` value and just pass the `timer_list` structure to the timeout function. The actual transition, though, is complicated by the existence of 800 or so `setup_timer()` call sites in the kernel now. Trying to change them all at once would not be anybody's idea of fun, so a phased approach is needed.

为了解决这些问题，库克一直致力于改变计时器界面。核心思想很简单：摆脱 数据值，只需将timer_list结构传递给超时函数即可。但是，由于内核中存在800个左右的setup_timer（）调用站点，实际的转换变得复杂了。试图一次性改变它们并不是任何人都有趣的想法，因此需要采用分阶段的方法。

> In this case, Cook has introduced a new function for the initialization of timers:

在这种情况下，Cook引入了一个新的定时器初始化函数：

    void timer_setup(struct timer_list *timer, void (*callback)(struct timer_list *),
		     unsigned int flags);

> For the time being, `timer_setup()` simply stores a pointer to `timer` in the `data` field. Note that the prototype of the callback has changed to expect the `timer_list` pointer.

目前，timer_setup（）只是在数据字段中存储一个指向计时器的指针 。请注意，回调的原型已更改为期望timer_list指针。

> With that function in place, calls to `setup_timer()` can be replaced at leisure, as long as each corresponding timer callback function is adjusted accordingly. For the most part, as can be seen in [this example](https://lwn.net/Articles/735892/), the changes are trivial. Many timer callbacks already were casting the `data` value to a pointer to the structure they needed; they just need a one-line change to obtain that from the `timer_list` pointer instead. A new `from_timer()` macro has been added to make those conversions a bit less verbose.

有了这个功能，只要相应地调整每个相应的定时器回调函数，就可以在闲暇时替换对setup_timer（）的调用。在大多数情况下，如本例所示，这些变化是微不足道的。许多计时器回调已经将数据值转换为指向它们所需结构的指针; 他们只需要进行一行更改即可从timer_list指针获取。添加了一个新的 from_timer（）宏，使这些转换不那么冗长。

> The addition of `timer_setup()` was merged just prior to the 4.14-rc3 release — rather later in the release cycle than one would ordinarily expect to see the addition of new interfaces. The purpose of this timing was clear enough: it clears the way for the conversion of all of those `setup_timer()` calls, a task which, it is hoped, will be completed for the 4.15 kernel release. Once that is done, the underlying implementation can be changed to drop the `data` value and the `setup_timer()` interface can be removed entirely. At the end, the kernel will be equipped with a timer mechanism that is a little bit more efficient, more like other in-kernel APIs, and easier to secure.

添加timer_setup（）在4.14-rc3发布之前合并 - 而不是在发布周期的后期，而不是通常期望看到添加新接口。这个时间的目的很明确：它清除了所有这些setup_timer（）调用的转换方式，希望这个任务能够在4.15内核版本中完成。完成后，可以更改底层实现以删除数据值，并且 可以完全删除setup_timer（）接口。最后，内核将配备一个更高效的计时器机制，更像其他内核API，更容易保护。

**请点击 [LWN 中文翻译计划](/lwn)，了解更多详情。**

[1]: https://elixir.bootlin.com/linux/v2.6.23/source/Documentation/sched-design-CFS.txt
[2]: https://elixir.bootlin.com/linux/v2.6.23/source/include/linux/sched.h#L896
[3]: https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/sched.h#L873
[4]: https://elixir.bootlin.com/linux/v2.6.28/source/Documentation/scheduler/sched-design-CFS.txt
[5]: https://elixir.bootlin.com/linux/v2.6.23/source/kernel/sched_fair.c#L979
[6]: https://kernelnewbies.org/Linux_2_6_23#The_CFS_process_scheduler
[7]: https://lwn.net/Articles/224865/
[8]: https://lwn.net/Articles/230500/
[9]: https://lwn.net/Articles/230752/
[10]: https://lwn.net/Articles/184495/
[11]: https://lwn.net/Articles/109460/
[12]: https://lwn.net/Articles/230628/
[13]: https://lwn.net/Articles/230501/
