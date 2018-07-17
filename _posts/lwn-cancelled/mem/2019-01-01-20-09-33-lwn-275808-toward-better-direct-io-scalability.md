---
layout: post
author: 'Wang Chen'
title: "LWN 275808: xxx"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-275808/
description: "LWN xxx"
category:
  - 内存子系统
  - LWN
tags:
  - Linux
  - memory
---

> 原文：[Toward better direct I/O scalability](https://lwn.net/Articles/275808/)
> 原创：By corbet @ Mar. 31, 2008
> 翻译：By [unicornx](https://github.com/unicornx) of [TinyLab.org][1]
> 校对：By [xxx](https://github.com/xxx)

> Linux enthusiasts like to point out just how scalable the system is; Linux runs on everything from pocket-size devices to supercomputers with several thousand processors. What they talk about a little bit less is that, at the high end, the true scalability of the system is limited by the sort of workload which is run. CPU-intensive scientific computing tasks can make good use of very large systems, but database-heavy workloads do not scale nearly as well. There is a lot of interest in making big database systems work better, but it has been a challenging task. Nick Piggin appears to have come up with a logical next step in that direction, though, with a relatively straightforward set of core memory management changes.

Linux爱好者喜欢指出系统的可扩展性如何; Linux可运行在从袖珍设备到具有数千个处理器的超级计算机的所有设备上。他们谈论的更少一点是，在高端，系统的真正可扩展性受到运行的工作负载的限制。CPU密集型科学计算任务可以很好地利用非常大的系统，但数据库繁重的工作负载几乎不能扩展。人们对使大型数据库系统更好地工作很感兴趣，但这是一项具有挑战性的任务。然而，尼克·皮金（Nick Piggin）似乎已经朝着这个方向迈出了合乎逻辑的下一步，通过一系列相对简单的核心内存管理变革。

> For some time, Linux has supported direct I/O from user space. This, too, is a scalability technology: the idea is to save processor time and memory by avoiding the need to copy data through the kernel as it moves between the application and the disks. With sufficient programming effort, the application should be able to make use of its superior knowledge of its own data access patterns to cache data more effectively than the kernel can; direct I/O allows that caching to happen without additional overhead. Large database management systems have had just that kind of programming effort applied to them, with the result that they use direct I/O heavily. To a significant extent, these systems use direct I/O to replace the kernel's paging algorithms with their own, specialized code.

一段时间以来，Linux已经支持用户空间的直接I / O. 这也是一种可扩展性技术：通过避免在应用程序和磁盘之间移动时通过内核复制数据的需要，可以节省处理器时间和内存。通过充分的编程工作，应用程序应该能够利用其对自己的数据访问模式的高级知识来比内核更有效地缓存数据; 直接I / O允许在没有额外开销的情况下进行缓存。大型数据库管理系统只是应用了这种编程工作，结果是他们大量使用直接I / O. 在很大程度上，这些系统使用直接I / O将内核的分页算法替换为自己的专用代码。

> When the kernel is asked to carry out a direct I/O operation, one of the first things it must do is to pin all of the relevant user-space pages into memory and locate their physical addresses. The function which performs this task is get_user_pages():

当要求内核执行直接I / O操作时，必须做的第一件事就是将所有相关的用户空间页面固定到内存中并找到它们的物理地址。执行此任务的函数是get_user_pages（）：

	int get_user_pages(struct task_struct *tsk, 
	                   struct mm_struct *mm, 
	                   unsigned long start,
	                   int len,
	                   int write,
	                   int force,
	                   struct page **pages,
	                   struct vm_area_struct **vmas);

> A successful call to `get_user_pages()` will pin `len` pages into memory, those pages starting at the user-space address `start` as seen in the given `mm`. The addresses of the relevant `struct page` pointers will be stored in `pages`, and the associated VMA pointers in `vmas` if it is not NULL.

成功调用get_user_pages（）会将len页面固定到内存中，从用户空间地址开始的那些页面将从 给定的mm开始。相关结构页面指针的地址将存储在页面中，如果不是NULL，则相关的VMA指针将在vmas中存储。

> This function works, but it has a problem (beyond the fact that it is a long, twisted, complex mess to read): it requires that the caller hold `mm->mmap_sem`. If two processes are performing direct I/O on within the same address space - a common scenario for large database management systems - they will contend for that semaphore. This kind of lock contention quickly kills scalability; as soon as processors have to wait for each other, there is little to be gained by adding more of them.

这个功能有效，但它有一个问题（除了它是一个漫长的，扭曲的，复杂的混乱阅读的事实）：它要求调用者持有 mm-> mmap_sem。如果两个进程在同一地址空间内执行直接I / O（大型数据库管理系统的常见方案），它们将争用该信号量。这种锁争用很快就会导致可扩展性; 一旦处理器必须等待彼此，通过添加更多的处理器几乎没有什么可以获得的。

> There are two common approaches to take when faced with this sort of scalability problem. One is to go with more fine-grained locking, where each lock covers a smaller part of the kernel. Splitting up locks has been happening since the initial creation of the Big Kernel Lock, which is the definitive example of coarse-grained locking. There are limits to how much fine-grained locking can help, though, and the addition of more locks comes at the cost of more complexity and more opportunities to create deadlocks.

面对这种可伸缩性问题时，有两种常见的方法。一种是使用更精细的锁定，其中每个锁覆盖内核的较小部分。自最初创建Big Kernel Lock以来，已经发生了拆分锁定，这是粗粒度锁定的最终示例。但是，细粒度锁定可以提供多少限制，并且增加更多锁定是以更复杂的成本和更多创建死锁的机会为代价的。

> The other approach is to do away with locking altogether; this has been the preferred way of improving scalability in recent years. That is, for example, what all of the work around read-copy-update has been doing. And this is the direction Nick has chosen to improve `get_user_pages()`.

另一种方法是完全取消锁定; 这是近年来提高可扩展性的首选方法。也就是说，例如，read-copy-update的所有工作都在做什么。这是Nick选择改进get_user_pages（）的方向。

> Nick's core observation is that, when `get_user_pages()` is called on a normal user-space page which is already present in memory, that page's reference count can be increased without needing to hold any locks first. As it happens, this is the most common use case. Behind that observation, though, are a few conditions. One is that it is not possible to traverse the page tables if those tables are being modified at the same time. To be guaranteed that this will not happen, the kernel must, before heading into the page table tree, disable interrupts in the current processor. Even then, the kernel can only traverse the currently-running process's page tables without holding `mmap_sem`.

Nick的核心观察是，当在内存中已存在的普通用户空间页面上调用get_user_pages（）时，可以增加该页面的引用计数，而无需先保存任何锁定。碰巧，这是最常见的用例。然而，在这一观察的背后，有一些条件。一个是如果同时修改这些表，则无法遍历页表。为了保证不会发生这种情况，内核必须在进入页表树之前禁用当前处理器中的中断。即使这样，内核也只能遍历当前正在运行的进程的页表而不保留mmap_sem。

> Lockless operation also will not work whenever pages which are not "normal" are involved. Some cases - non-present pages, for example - are easily detected from the information found in the page tables themselves. But others, such as situations where the relevant part of the address space has been mapped onto device memory with `mmap()`, are not readily apparent by looking at the associated page table entries. In this case, the kernel must look back at the controlling `vm_area_struct` (VMA) structure to see what is going on - and that cannot be done without holding `mmap_sem`. So it looks like there is no way to find out whether lockless operation is possible without first taking the lock.

无论何时涉及非“正常”的页面，无锁操作也将不起作用。某些情况 - 例如，不存在的页面 - 可以从页面表本身中找到的信息中轻松检测到。但是其他的，例如地址空间的相关部分已经用mmap（）映射到设备存储器的情况，通过查看相关的页表条目并不是很明显。在这种情况下，内核必须回顾控制vm_area_struct（VMA）结构以查看正在发生的事情 - 如果不保存mmap_sem则无法完成 。所以看起来没有办法在没有先锁定的情况下找出无锁操作是否可行。

> The solution here is to grab a free bit in the page table entry. The PTE for a page which is present in memory holds the physical page frame address. In such addresses, the bottom 12 bits (for architectures using 4096-byte pages) will always be zero, so they can be dedicated to other purposes. One of them is used to indicate whether the page is present in memory at all; others indicate writability, whether it's a user-space page, whether it is dirty, etc. Nick's patch grabs one of the few remaining bits and calls it "PAGE_BIT_SPECIAL," indicating "special" pages. These are pages which, for whatever reason, do not have a readily-accessible `struct page` associated with them. Marking "special" pages in the page tables can help in a number of places; one of those is making it possible to determine whether lockless `get_user_pages()` is possible on a given page.

这里的解决方案是在页表条目中获取一个空闲位。存储在存储器中的页面的PTE保存物理页面帧地址。在这样的地址中，底部的12位（对于使用4096字节页面的体系结构）将始终为零，因此它们可以专用于其他目的。其中一个用于指示页面是否存在于内存中; 其他人表示可写性，无论是用户空间页面，是否脏，等等。尼克的补丁抓住剩下的几个位之一并称之为“ PAGE_BIT_SPECIAL ”，表示“特殊”页面。这些页面无论出于何种原因，都没有易于访问的结构页面与他们相关联。在页面表中标记“特殊”页面可以在许多地方提供帮助; 其中一个是可以确定给定页面上是否可以进行无锁 get_user_pages（）。

> Once these pages are properly marked in the page tables, it is possible to write a function which makes a good attempt at a lockless `get_user_pages(`). Nick's [proposal](http://lwn.net/Articles/275724/) is called `fast_gup()`:

一旦在页表中正确标记了这些页面，就可以编写一个能够很好地尝试无锁get_user_pages（）的函数 。尼克的提议称为 fast_gup（）：

	int fast_gup(unsigned long start, int nr_pages, 
	             int write, struct page **pages);

> This function has a much simpler interface than `get_user_pages()` because it does not handle many of the cases that `get_user_pages()` can deal with. It only works with the current process's address space, and it cannot return pointers to VMA structures. But it *can* iterate through a set of page tables, testing each page for presence, writability, and "non-specialness," and incrementing each page's reference count (thus pinning it into physical memory) in the process. If it works, it's very fast. If not, it undoes things then falls back to `get_user_pages()` to do things the slow, old-fashioned way.

此函数具有比get_user_pages（）更简单的接口， 因为它不处理get_user_pages（） 可以处理的许多情况。它只适用于当前进程的地址空间，并且不能返回指向VMA结构的指针。但它 *可以* 遍历一组页表，测试每个页面的存在性，可写性和“非特殊性”，并在此过程中递增每个页面的引用计数（从而将其固定到物理内存中）。如果它有效，那就非常快。如果没有，它会撤消事情，然后回到 get_user_pages（）以缓慢，老式的方式做事。

> How much is this worth? Nick claims a 10% performance improvement running "an OLTP workload" (one of those unnameable benchmark suites, perhaps) using IBM's DB2 DBMS system on a two-processor (eight-core) system. The performance improvement, he says, may be greater on larger systems. But even if it remains at "only" 10%, this work is a clear step in the right direction for this kind of workload.

这值多少钱？Nick声称使用IBM的DB2 DBMS系统在双处理器（八核）系统上运行“OLTP工作负载”（可能是其中一个不可命名的基准测试套件），性能提高了10％。他说，在大型系统上，性能的提升可能更大。但即使它仍然只有“10％”，这项工作也是朝着这种工作量正确方向迈出的明确一步。

> [Update: this interface was merged for the 2.6.27 kernel; the name was changed to `get_user_pages_fast()` but it is otherwise the same.]

[ 更新：此接口已合并为2.6.27内核; 名称已更改为get_user_pages_fast（）但其他方面相同。]

[1]: http://tinylab.org
