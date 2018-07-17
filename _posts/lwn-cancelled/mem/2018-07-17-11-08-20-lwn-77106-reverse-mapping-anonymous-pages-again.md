---
layout: post
author: 'Wang Chen'
title: "LWN 77106: xxx"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-77106-reverse-mapping-anonymous-pages-again/
description: "LWN 文章翻译，xxx"
category:
  - 内存子系统
  - LWN
tags:
  - Linux
  - memory
---

> 原文：[Reverse mapping anonymous pages - again](https://lwn.net/Articles/77106/)
> 原创：By corbet @ Mar. 24, 2004
> 翻译：By [unicornx](https://github.com/unicornx) of [TinyLab.org][1]
> 校对：By [xxx](https://github.com/xxx)

> Two weeks ago, this page [described Andrea Arcangeli's "anon_vma" work](https://lwn.net/Articles/74295/) in some detail. This work, remember, is an attempt to improve memory scalability in the kernel by eliminating the reverse mapping ("rmap") chains used to find page table entries which reference a given page. The rmap chains can use significant amounts of low memory and can slow down `fork()` calls, so this work is of interest.

两周前，这个页面描述了 Andrea Arcangeli 所实现的 “anon_vma” 的一些细节。这项工作消除了原有反向映射机制（reverse mapping，下文简称 "rmap"）中使用的链表结构（这些链表在 rmap 中用于反向查找映射物理页面的页表条目），而这些链表结构会大量消耗低端内存并降低 `fork
()` 的执行速度。这项工作使内核可以支持更大规模的内存容量，是非常有意义的。
 
> Andrea has continued pushing the anon_vma effort through a series of kernel tree releases. The latest, [2.6.5-rc2-aa2](https://lwn.net/Articles/77050/), solves some of the remaining problems and comes with this statement:

>     The next target is the merging of the prio_tree, but that will be a separated patch. After that this whole thing should be mergeable into mainline.

Andrea 对 anon_vma 的改进经历了多个内核版本的升级。最新修改合入了 [2.6.5-rc2-aa2](https://lwn.net/Articles/77050/) 并解决了一些剩余的问题，他在提交的补丁中声明如下：

    我的下一个目标是加入 prio_tree 的功能，但这将是一个单独的补丁。这项工作完成之后，整个修改都可以合并入主线。

> (The prio_tree reference is about Rajesh Venkatasubramanian's [priority tree patch](https://lwn.net/Articles/76621/) which speeds the search for interesting virtual memory areas when a page is mapped a large number of times).

（所谓 prio_tree 是指 Rajesh Venkatasubramanian 开发的 [priority tree patch](https://lwn.net/Articles/76621/) 补丁，当一个物理页被多次映射时，该算法会加速搜索所有相关的虚拟内存线性空间）。

> Andrea's work is proceeding nicely, but it's worth noting that anon_vma is not the only approach to the implementation of an object-based reverse mapping scheme for anonymous memory. There is competition in the form of "anonmm" by Hugh Dickins. Hugh has recently reworked the patch and posted it for comments; interested parties can find this (multi-part) posting in the "patches" section below.

Andrea 的工作进展顺利，但值得注意的是 anon_vma 并不是对匿名内存实现基于对象的反向映射方案的唯一选择。Hugh Dickins 提供了另一个以 “anonmm” 为命名的解决方案。Hugh 最近重新修改并发布了该补丁供大家审阅；感兴趣的人士可以在下面的 “patches” 章节部分找到这个（由多个部分组成）的帖子。

> The anon_vma patch works by creating a linked list of virtual memory areas (VMAs) which reference a given page. The anonmm patch, instead, creates a connection between an anonymous page and the mm_struct structures which reference it. The mm_struct is the top-level structure used to manage a process's virtual address space; it contains pointers to all of the process's VMAs and page tables, along with various bits of locking and housekeeping information. If you have a pointer to a process's mm_struct and a virtual address, you can quickly walk the page tables and determine whether the given address is a reference to a specific page.

anon_vma 补丁通过为给定的物理页创建所有具备映射关系的虚拟内存区域（virtual memory areas，简称 VMA）的链表来工作。相反，anonmm 补丁则建立了匿名页和引用它的所有 `mm_struct` 结构之间的联系。`mm_struct` 是用于管理一个进程的虚拟地址空间的顶层结构；它包含指向所有进程的 VMA 和页表的指针，以及各种锁和管理结构信息。假定您获得了一个指向进程的 `mm_struct` 的指针以及虚拟地址信息，就可以快速遍历页表并确定给定地址是否映射了特定的物理页。

> Most of the object-based reverse mapping has worked with the VMA structure. When performing reverse mapping of file-backed pages, use of the VMA structure is unavoidable; if multiple processes have mapped the file into their address spaces, each process likely has a different virtual address for the same page. The VMA structure contains the necessary information to determine which virtual address each process will have for a specific offset within a file. Once that address is found, the page of interest can be unmapped from that process's address space.

大多数基于对象的反向映射都适用于VMA结构。当执行文件支持页面的反向映射时，使用VMA结构是不可避免的; 如果多个进程已将文件映射到其地址空间，则每个进程可能具有相同页面的不同虚拟地址。VMA结构包含必要的信息，以确定每个进程对文件中的特定偏移量将具有哪个虚拟地址。找到该地址后，可以从该进程的地址空间取消映射感兴趣的页面。

> Anonymous pages are different from file-backed pages, however; they are only shared between processes when a process forks (and, even then, it's a copy-on-write sharing). That means that, with one exception that we'll get to, shared anonymous pages have the same virtual address in every process. Thus, if you can track an anonymous page's virtual address and the processes which share that page, you can quickly find all of the page table entries referencing the page.

但是，匿名页面与文件支持的页面不同; 它们仅在进程分叉时在进程之间共享（即使这样，它也是写时复制共享）。这意味着，除了我们将要达到的一个例外，共享匿名页面在每个进程中都具有相同的虚拟地址。因此，如果您可以跟踪匿名页面的虚拟地址以及共享该页面的进程，则可以快速找到引用该页面的所有页表条目。

> The anonmm patch takes advantage of this fact. An anonymous page's virtual address is stored in the `index` field of the page structure. This field is normally used to give a page's offset within the file that backs it, but, since anonymous pages have no backing file, the field is available for this use. Hugh's patch then creates a new `anonmm` structure which is used to create a linked list of `mm_struct` structures; a pointer to this list is also stored in the `page` structure. The resulting data structure looks roughly like this:

anonmm补丁利用了这一事实。匿名页面的虚拟地址存储在页面 结构的索引字段中。此字段通常用于在支持它的文件中提供页面偏移量，但是，由于匿名页面没有后备文件，因此该字段可用于此用途。Hugh的补丁然后创建一个新的 anonmm结构，用于创建mm_struct结构的链接列表 ; 指向此列表的指针也存储在页面结构中。结果数据结构大致如下所示：

![cheesy anonmm diagram](https://static.lwn.net/images/ns/anonmm.png)
 
> With this structure in place, the kernel can follow the pointers to quickly find the page tables referencing a given anonymous page. This approach, in theory, should be a little simpler and faster than the anon_vma technique; a process may have several VMAs for anonymous memory areas, but it will never have more than one mm_struct.

有了这个结构，内核可以按照指针快速找到引用给定匿名页面的页表。理论上，这种方法应该比anon_vma技术更简单，更快速; 一个进程可能有多个用于匿名内存区域的VMA，但它永远不会有多个mm_struct。

> There is one little problem with this whole scheme. It depends on the fact that every process has the same virtual address for a given, shared anonymous page. What happens when some wiseass process comes along and moves a chunk of anonymous memory with mremap()? At that point, the memory has a new address, and the anonmm algorithm will be unable to find it. Hugh's solution for this problem is to simply copy the pages being remapped. They are copy-on-write pages, so making copies will not create any correctness issues. The copying could be expensive - it may involve swapping in a number of pages so that they can be copied - but remapping of anonymous memory should be a sufficiently rare operation that a performance hit should not be a problem.

整个计划有一个小问题。这取决于每个进程对于给定的共享匿名页面具有相同的虚拟地址这一事实。当一些明智的进程出现并使用mremap（）移动一大块匿名内存时会发生什么？此时，内存有一个新地址，anonmm算法将无法找到它。Hugh针对此问题的解决方案是简单地复制正在重新映射的页面。它们是写时复制页面，因此制作副本不会产生任何正确性问题。复制可能很昂贵 - 它可能涉及交换多个页面以便可以复制它们 - 但重新映射匿名内存应该是一个非常罕见的操作，性能命中应该不是问题。

> Which scheme is truly faster? Martin Bligh has posted [a set of benchmarks](https://lwn.net/Articles/77140/) showing that, while both reverse mapping approaches are significantly faster than the mainline kernel, neither is obviously faster than the other. Andrea's work is marginally ahead in more tests than Hugh's, but, overall, the two produce roughly equivalent results. So, if one of these implementations does find its way into the 2.6 kernel, it will have to be chosen for reasons other than performance. Either that, or it will be some combination of the two; Andrea and Hugh are actively discussing ideas, so that sort of combination could happen.

哪种方案真的更快？Martin Bligh发布了一组基准测试，虽然两种反向映射方法都明显快于主线内核，但两者都没有明显快于另一种。Andrea的工作在比Hugh更多的测试中略微提前，但总的来说，两者产生大致相同的结果。因此，如果其中一个实现确实进入2.6内核，则必须选择性能以外的其他原因。要么是，要么是两者的某种组合; 安德里亚和休正在积极讨论各种想法，以便实现这种组合。

[1]: http://tinylab.org
