---
layout: post
author: 'Wang Chen'
title: "LWN 594725: xxxx"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-594725/
description: "LWN 文章翻译，xxxx"
category:
  - 内存子系统
  - LWN
tags:
  - Linux
  - memory
---

> 原文：[Avoiding memory-allocation deadlocks](https://lwn.net/Articles/594725/)
> 原创：By Neil Brown @ April 16, 2014
> 翻译：By [unicornx](https://github.com/unicornx) of [TinyLab.org][1]
> 校对：By [xxx](https://github.com/xxx)

> There is a saying that you need to spend money to make money, though this apparent paradox is easily resolved with a start-up loan and the discipline of balancing expenses against income. A similar logic applies to the management of memory in an operating system kernel such as Linux: sometimes you need to allocate memory to free memory. Here, too, discipline is needed, though the typical consequences of not being sufficiently careful is not bankruptcy but rather a deadlock.

有人说，不花钱怎么赚钱，虽然这听上去有点像悖论但解决的方法其实也很简单，就是先贷款获得启动资金，然后严格控制财务上的收支平衡即可。类似的逻辑同样适用于操作系统内核（如 Linux）中的内存管理：有时为了释放内存却不得不先申请分配一点内存。在这里，也需要遵守严格的规则进行处理，否则一不小心，典型的后果不是破产而是系统死锁（deadlock）。

> The history of how the Linux kernel developed its balance between saving and spending is interesting as a microcosm of how Linux development proceeds, and useful in understanding how to handle future deadlocks when they occur. A good place to start this history is in early 1998 with the introduction of `__GFP_IO` in Linux 2.1.80pre3.

作为整个 Linux 开发过程的缩影，了解一下内核在如何管理内存分配，并保证收支平衡上的历史发展一定非常有趣，并且有助于我们在将来遇到类似的死锁问题时知道如何解决它们。让我们穿越到 1998 年初，那时候的 Linux 版本还是 2.1.80pre3，在这个版本中首次引入了 `__GFP_IO` 标志宏定义，就从这里开始我们的回顾之旅吧。

## `__GFP_IO` in 2.1.80pre3

> Any memory allocation request in Linux includes a `gfp_t` argument, which is a set of flags to guide how the `get_free_page()` function can go about locating a free page. [2.1.80pre3](https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=472bbf0af5bc9f7c933a5d3212b0d765176e728a) marks a change in this argument's type; it went from being a simple enumerated type to being a bitmask. The concepts embodied in each flag were present previously, but this is the first time that they could be explicitly identified.

当前 Linux 中所有内存分配操作都会涉及一个类型为 `gfp_t` 参数，该参数是一组比特位标志，用于指示 `get_free_page()` 函数分配一个空闲的物理页框。从内核版本 [2.1.80pre3](https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=472bbf0af5bc9f7c933a5d3212b0d765176e728a) 开始我们可以发现该参数类型的变化；从简单的枚举类型变为以比特位掩码的方式。这些标志所包含的含义在早先的版本中都是有定义的，但从这个版本开始我们第一次明确使用比特位的方式来定义它们。

> `__GFP_IO` was one of the new flags. If it was set, then `get_free_pages()` was allowed to call `shm_swap()` to write some pages out to swap. If `shm_swap()` needed to allocate any `buffer_head` structures to complete the writeout, it would be careful not to set `__GFP_IO`. Without this protection, an infinite recursion could easily happen, which would quickly exhaust the stack and cause the kernel to crash.

在该版本中新增了一个标志 `__GFP_IO`。如果指定了该标志位，则 `get_free_pages()` 函数中将被允许调用 `shm_swap()` 将一些页框换（swap）出。但要注意的是，如果 `shm_swap()` 函数在执行过程中需要分配 `buffer_head` 结构体来完成写出操作，那指定 `__GFP_IO` 就会出问题。因为一旦设置了 `__GFP_IO`，分配内存过程中很可能会发生无限递归，导致耗尽内核栈并导致内核崩溃。

> We have `__GFP_IO` in the kernel today, but, despite having the same name, it is a different flag. Having been introduced for 2.1.80, the original `__GFP_IO` was removed in 2.1.116, to be replaced with...

当前最新版本的内核中依然存在 `__GFP_IO`，但是，尽管名称相同，含义已经完全不同。自从该标志位在 2.1.80 版本中被引入后，在 2.1.116 版本中曾经被删除，并采用了新的方式来替换它 ......

## `PF_MEMALLOC` in 2.1.116

> In the [distant past](https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=ac995a26c87ac75983960cbe4085a77b6bbe3e4d) (August 1998), we did not have change logs of nearly the quality that we have today, so an operating-system archaeologist is left to guess at the reasons for changes. All we can be really sure of is that the (per-request) `__GFP_IO` flag to `get_free_page()` disappeared, and a new per-process flag called `PF_MEMALLOC` appeared to take over the task of avoiding recursion. One clear benefit of this change is that it is more focused in addressing one particular issue: recursion is clearly a per-process issue and so a per-process flag is fitting. Previously, many memory allocation sites would avoid `__GFP_IO` when they didn't really need to, just in case. Now each call site doesn't need to worry about the problem of recursion; that concern is addressed precisely where it is needed.

在遥远的[过去](https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=ac995a26c87ac75983960cbe4085a77b6bbe3e4d)（1998年8月），我们没有能和当前媲美的修改日志管理系统，因此当我试图考证一些修改的原因时不得不加入自己的猜测。我们所能确定的是，请求分配内存时传递给 `get_free_page()` 函数的 `__GFP_IO` 标志被删除了，取而代之的是增加了一个名为 `PF_MEMALLOC` 的进程标志。同样是用于解决避免递归调用的问题，新方法的一个明显好处是它更贴近我们所要解决的问题：而递归显然是一个针对每个进程的问题，因此在进程标志上增加选项更合适。以前，除非万不得已，内核中许多申请内存分配的地方都会尽量避免使用 `__GFP_IO`，而目的仅仅是为了以防万一（指出现递归的问题）。现在则没有这种担心的必要；内核会在更恰当的地方考量这个问题。

> The code comments here highlight an important aspect of memory allocation:

下面的代码注释强调了内存分配中的这个重要方面：

	 * The "PF_MEMALLOC" flag protects us against recursion:
	 * if we need more memory as part of a swap-out effort we
	 * will just silently return "success" to tell the page
	 * allocator to accept the allocation.

> When possible, `get_free_page()` will just pluck a page off the free list and return it as quickly as it can. When that is not possible, it does not satisfy itself with freeing just one page, but will try to free quite a few, to save work next time. Thus, it is re-stocking that startup loan. A particular consequence of `PF_MEMALLOC` is that the memory allocator won't try too hard to gets lots of pages; it will make do with what it has.

（译者注，以下描述参考 2.1.116 版本的 [`root/mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/tree/mm/page_alloc.c?id=ac995a26c87ac75983960cbe4085a77b6bbe3e4d) 中的 `__get_free_pages()` ，本文提到的 `get_free_page()` 内部最终还是调用的该函数，但只是申请分配一个页框）如果还有空闲的内存，`get_free_page()` 会分配一个页框后立即返回。如果发现内存不足（即对应代码 `if(freepages.min > nr_free_pages)`），它会尝试释放（free）多个页面而不仅仅只是一个，以确保未来可以有足够的内存可以使用。这么做相当于提前准备好贷款等待下次使用。而如果设置了 `PF_MEMALLOC` 则内存分配器不会尝试释放大量的物理页框；它将直接使用当前还可用的内存（译者注，参考[root/mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/tree/mm/vmscan.c?id=ac995a26c87ac75983960cbe4085a77b6bbe3e4d) 中的 `try_to_free_pages()`，但这里我有个疑问就是感觉此处代码的逻辑和文章的描述是颠倒的，`PF_MEMALLOC` 的标志位设置的效果怎么反而是会去尝试释放内存呢，年代久远不可考）。

> This means that processes with the `PF_MEMALLOC` flag set will have access to the last dregs of free memory, while other processes will need to go out and free up lots of memory first before they can use any. This property of `PF_MEMALLOC` is still present and somewhat more formal in the current kernel. The memory allocator has a concept of "watermarks" such that, if the amount of free memory is below the chosen watermark, the allocator will try to free more memory rather than return what it has. Different `__GFP` flags can select different watermark levels (min, low, high). `PF_MEMALLOC` causes all watermarks to be ignored; if any memory is available at all, it will be returned.

这意味着设置了 `PF_MEMALLOC` 标志的进程可以访问最后一个可用内存，而其他进程需要先释放大量内存之后才能继续分配内存。`PF_MEMALLOC` 的这个属性 在当前内核中仍然存在，而且意义更明确。在内存分配器中有一个所谓 “水位线” （"watermarks"）的概念，如果当前可用内存量低于所指定的 “水位线”，则分配器将尝试释放更多内存而不再继续分配内存。我们可以指定不同的 `__GFP` 标志来选择不同的“水位线”级别（最小值（min），低（low），高（high））。 而指定了 `PF_MEMALLOC` 则导致分配器忽略所有的“水位线”；只要还有内存可用就返回。

> `PF_MEMALLOC` effectively says "It is time to stop saving and start spending, or we'll have no product to sell". In consequence of this, `PF_MEMALLOC` is now used more broadly than just for avoiding recursion (though it still has that role). Several kernel threads, such as those for `nbd`, the network block device, `iscsi_tcp`, and the MMC card controller, all set `PF_MEMALLOC`, presumably so they can be sure to get memory whenever they are called upon to write out a page of memory (so it can be freed).

指定 `PF_MEMALLOC` 相当于告诉内核 “现在是时候停止储蓄而要开始消费了，否则我们没有产品可以出售”。因此，`PF_MEMALLOC` 不是仅仅用于避免递归（尽管它仍然具备该功能），而是可以用于更广泛的场景。一些内核线程，例如用于 nbd，网络块设备，`iscsi_tcp` 和 MMC 卡控制器，都会设置 `PF_MEMALLOC` 标志，大概这样可以确保在每次调用写出一页内存时获取内存（所以它可以被释放）。

> In contrast, the MTD driver (which manages NAND flash and has a similar role to the MMC card driver) stopped using the `PF_MEMALLOC` flag in 2.6.33 with a comment suggesting it was an inappropriate usage. Whether the other uses in the kernel are still justified is a question too deep for our present discussion.

相比之下，MTD驱动程序（管理NAND闪存并具有与MMC卡驱动程序类似的作用）停止使用 2.6.33中的PF_MEMALLOC标志，并发表评论表明这是一种不恰当的用法。对于我们目前的讨论来说，内核中的其他用途是否仍然合理是一个问题。

## `__GFP_IO` in 2.2.0pre6

> When `__GFP_IO` [reappears](https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=70c27ee94003b5e3741c5d36f5a84ac6cc81ae82) it has a similar purpose as the original, but for an importantly different reason. To understand that reason, it suffices to look at a comment in the code:

当__GFP_IO 重新出现时， 它具有与原始目的相似的目的，但出于一个非常不同的原因。要理解这个原因，只需查看代码中的注释即可：

	/*
	 * Don't go down into the swap-out stuff if
	 * we cannot do I/O! Avoid recursing on FS
	 * locks etc.
	 */

> The concern here still involves recursion, but it also involves locks, such as the per-inode mutex, the page lock, or various others. Calling into a filesystem to write out a page may require taking a lock. If any such lock is held when allocating memory then it is important to avoid calling into any filesystem code that might try to acquire the same lock. In those cases, the code must be careful not to pass `__GFP_IO`; in other cases, it is perfectly safe to include that flag.

这里的问题仍然涉及递归，但它也涉及锁，例如每个inode互斥锁，页锁或其他各种锁。调用文件系统来写出页面可能需要锁定。如果在分配内存时保持任何此类锁定，则避免调用可能尝试获取相同锁定的任何文件系统代码是很重要的。在这些情况下，代码必须小心不要传递__GFP_IO ; 在其他情况下，包含该标志是完全安全的。

> So while `PF_MEMALLOC` avoids the specific recursion of `get_free_page()` calling into `get_free_page(`), `__GFP_IO` is more general and prevents any function holding a lock from calling, through `get_free_page()`, into any other function which might want that lock. The risk here isn't exhausting the stack as with `PF_MEMALLOC`; the risk is a deadlock.

因此，尽管PF_MEMALLOC避免的具体递归 get_free_page（）调用到get_free_page（） ，__GFP_IO比较一般，并防止持有锁的任何功能与调用，通过 get_free_page（），在其中可能希望该锁的其它任何功能。这里的风险并不像PF_MEMALLOC那样耗尽堆栈; 风险是僵局。

> One might wonder why a `GFP` flag was used for this rather than a process flag, which would effectively say "I am holding a filesystem lock", given that the previous experience with `__GFP_IO` wasn't a success. Like many software designs, it probably just "seemed like a good idea at the time".

人们可能想知道为什么GFP标志被用于此而不是进程标志，这将有效地说“我持有文件系统锁”，因为之前使用__GFP_IO的经验并不成功。像许多软件设计一样，它可能只是“当时似乎是一个好主意”。

## `__GFP_FS` in 2.4.5.8

> This flag started life named `__GFP_BUFFER` in 2.4.5.1, but didn't really work properly until 2.4.5.8 when it was renamed to `__GFP_FS`. Apparently there was a [thinko](http://git.kernel.org/cgit/linux/kernel/git/tglx/history.git/commit/?id=75b566af5cc6f64f9ab5b66608ff8ce18098a2b4) in the original design, which required not only a range of code changes, but also a new name.

这个标志在2.4.5.1中开始命名为__GFP_BUFFER，但直到2.4.5.8重命名为__GFP_FS时才真正起作用。显然 原始设计中有一个thinko，它不仅需要一系列代码更改，还需要一个新名称。

> `__GFP_FS` effectively split some functionality away from `__GFP_IO` so that where there was one flag, now there were two. Only three combinations of the two were expected: neither, both, or the new possibility of just the `__GFP_IO` flag being set. This would allow buffers that were already prepared to be written out, but would prohibit any calls into filesystems to prepare those buffers. I/O activity would be permitted, but filesystem activity would not.

__GFP_FS有效地将一些功能从__GFP_IO中分离出来， 以便在有一个标志的地方，现在有两个。预计只有三种组合：既没有，也没有，或者只是设置了__GFP_IO标志的新可能性。这将允许已经准备好的缓冲区被写出，但是会禁止对文件系统的任何调用来准备这些缓冲区。允许I / O活动，但文件系统活动不允许。

> Presumably, the fact that `__GFP_IO` previously had such a broad effect was harming performance, in that it had to be excluded in places where some I/O was still possible. Refining the rules by adding a new flag led to more flexibility, and so fewer impediments to performance.

据推测，__GFP_IO以前具有如此广泛的影响这一事实正在损害性能，因为它必须被排除在仍然可能存在某些I / O的地方。通过添加新标志来改进规则可以提高灵活性，从而减少对性能的阻碍。

## `PF_FSTRANS` in 2.5.36

> This new process flag appeared when [XFS support](https://git.kernel.org/cgit/linux/kernel/git/tglx/history.git/commit/?id=ef5cc2fd95520561f7e3c8c49b809000dee033ba) was merged into Linux in late 2002. Its purpose was to indicate that a filesystem transaction (hence the name) was being prepared, meaning that any write to the filesystem would likely block until the transaction processing was complete. The effect of this flag was to exclude `__GFP_FS` from any memory allocation request which happened while `PF_FSTRANS` was set, or at least any request from within the XFS code. Other requests would not be affected, but then other code that allocated memory would be unlikely to be called while the flag was set.

这个新的进程标志出现在2002年末XFS支持合并到Linux中。它的目的是指示正在准备文件系统事务（因此名称），这意味着在事务处理完成之前，对文件系统的任何写入都可能会阻塞。此标志的作用是从设置PF_FSTRANS时发生的任何内存分配请求中排除__GFP_FS，或者至少从XFS代码中发出任何请求。其他请求不会受到影响，但是在设置标志时，不太可能调用分配内存的其他代码。

> Another way to see this flag is that, in the same way that the original `__GFP_IO` was converted to `PF_MEMALLOC`, now `__GFP_FS` is being converted to a process flag, too. In this case, the conversion is not complete, though.

另一种查看此标志的方法是，与原始__GFP_IO转换为PF_MEMALLOC的方式相同，现在__GFP_FS也正在转换为进程标志。在这种情况下，转换不完整。

> Back in the halcyon days of 2.1.116, removing a flag like `__GFP_IO` was quite straightforward — there were few users and the implications of the change could be easily understood. In the more complex times of 2.5.36, such a step would be far from easy. Carefully adding new functionality is one thing, removing something that is entrenched is quite another, as we have seen with the [Big Kernel Lock](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=4ba8216cd90560bc402f52076f64d8546e8aefcb) and the [`sleep_on()` interface](https://lwn.net/Articles/593670/). Allowing either the new flag or the absence of the old to have the same effect is not a big cost and it was best to leave things that were working alone.

回到2.1.116的平静时期，删除像__GFP_IO这样的标志 非常简单 - 用户很少，并且可以很容易地理解变更的含义。在2.5.36的更复杂时期，这样的步骤将远非易事。仔细添加新功能是一回事，删除根深蒂固的东西是另一回事，正如我们在Big Kernel Lock和sleep_on（） 接口中看到的 那样。允许新标志或旧标志的缺席具有相同的效果并不是一个很大的成本，最好留下单独工作的东西。

> Skipping ahead of ourselves a little to [3.6-rc1](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=5cf02d09b50b1e), the `PF_FSTRANS` flag also gets used by NFS. Rather than setting it during a transaction, NFS sets it while constructing and transmitting an RPC request onto the network, so the name is now slightly less appropriate. Also the effect of the flag on NFS is not exactly to clear `__GFP_FS`, but simply to avoid a call to transmit a `COMMIT` request inside `nfs_release_page()`, which is also avoided if `__GFP_FS` is missing. This is a superficially different usage than the usage by XFS, but it has a generally similar effect for a generally similar reason. Modifying the flag to have a more global effect of clearing `GFP_FS` and maybe renaming it to `PF_MEMALLOC_NOFS` might not be a bad idea.

在我们自己略微超过3.6-rc1时，PF_FSTRANS 标志也被NFS使用。NFS不是在事务期间设置它，而是在构建RPC请求到网络时设置它，因此现在名称稍微不合适。此外，标志对NFS的影响并不完全清除__GFP_FS，而只是为了避免调用在nfs_release_page（）中发送COMMIT请求 ，如果缺少__GFP_FS，也可以避免这种情况。这与XFS的使用情况略有不同，但由于一般类似的原因，它具有类似的效果。修改标志以更清晰地清除GFP_FS并且可能将其重命名为PF_MEMALLOC_NOFS可能不是一个坏主意。

## `set_gfp_allowed_mask()` in 2.6.34

> This function actually [appeared](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=7e85ee0c1d15ca5f8bff0f514f158eba1742dd87) in [2.6.31](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=dcce284a259373f9e5570f2e33f79eca84fcf565), but becomes more interesting in 2.6.34.

这个功能实际上出现 在2.6.31中，但在2.6.34中变得更有趣。

> `gfp_allowed_mask` is a global variable holding a set of `GFP` flags which are allowed to be honored — all others are ignored. In particular, `__GFP_FS`, `__GFP_IO`, and `__GFP_WAIT` (which generally allows `get_free_page()` to wait for memory to be freed by other processes) are sometimes disabled via this mechanism. Thus it is a bit like `PF_FSTRANS`, except that it affects more processes and disables more flags.

gfp_allowed_mask是一个全局变量，包含一组GFP标志，允许遵守这些标志 - 所有其他标志都被忽略。特别是，__GFP_FS，__ GFP_IO和__GFP_WAIT（通常允许get_free_page（）等待内存被其他进程释放）有时会通过此机制禁用。因此它有点像 PF_FSTRANS，除了它影响更多进程并禁用更多标志。

> `gfp_allowed_mask` came about while providing support for `kmalloc()` earlier in the boot process. During early boot, interrupts are still disabled and any attempt to allocate memory with `__GFP_WAIT` or certain other flags can trigger a warning from the `lockdep` checker. It would be surprising if memory were so tight during boot that the allocator actually needed to wait, but getting rid of warnings is generally a good thing, so `gfp_allowed_mask` was initialized to exclude the three flags mentioned, and these were added back in once the boot process was complete.

gfp_allowed_mask是 在引导过程早期为kmalloc（）提供支持时 发生的。在早期启动期间，中断仍然被禁用，任何使用__GFP_WAIT或某些其他标志分配内存的尝试都可以触发lockdep 检查程序的警告。令人惊讶的是，如果内存在启动期间如此紧张以至于分配器实际上需要等待，但是摆脱警告通常是件好事，因此gfp_allowed_mask被初始化以排除所提到的三个标志，并且这些在引导时被添加回来过程完成了。

> One thing we have learned over the years is that boot isn't as special as we sometimes think: whether it is suspend and resume, or hotplug hardware which teaches us this, it seems to be a lesson we keep finding new perspectives on. In that light, it is perhaps unsurprising that, in [2.6.34](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=452aa6999e6703ffbddd7f6ea124d3968915f3e3), the use of this mask was extended to cover suspend and resume (though an [early version](https://lkml.org/lkml/2009/6/12/82) of the original patch did mention the importance of suspend).

我们多年来学到的一件事是，开机并不像我们有时想的那么特别：无论是暂停和恢复，还是热插拔硬件教会我们这一点，这似乎是我们不断寻找新观点的一课。从这个角度来看，可能不足为奇的是，在2.6.34中，这个面具的使用被扩展到包括暂停和恢复（虽然原始补丁的早期版本确实提到了暂停的重要性）。

> In the case of memory allocation deadlocks, the suspend case is more significant than the boot case. During boot there is usually lots of free memory — not so during suspend, when we may well be short of memory. It wasn't warnings that prompted this change, but genuine deadlocks.

在内存分配死锁的情况下，挂起情况比引导情况更重要。在启动过程中通常会有大量的空闲内存 - 暂停期间不会这样，因为我们可能内存不足。这并不是警告促使这种变化，而是真正的僵局。

> Suspend and resume are largely orderly processes, with devices being put to sleep in sequence, and then woken again in the reverse sequence. So it would not be enough just for block devices to avoid using `__GFP_IO` (which they already do). Rather, every driver must avoid the `__GFP_IO` flag, and others, as the target block device of some write request, might be sequenced with this driver so that it is already asleep, and will not awake before this one is completely awake.

暂停和恢复基本上是有序的过程，设备按顺序进入睡眠状态，然后以相反的顺序再次唤醒。因此，仅阻止块设备避免使用__GFP_IO（他们已经这样做）是不够的。相反，每个驱动程序必须避免使用__GFP_IO标志，而其他驱动程序可能会使用此驱动程序对某个写入请求的目标块设备进行排序，以使其已经处于睡眠状态，并且在此驱动程序完全清醒之前不会唤醒。

> Having a system-wide setting to disable these flags may be a bit excessive — just the process which is sequencing suspend might be sufficient — but it is certainly an easy fix and, as it cannot affect normal running of the system, it is thus a safe fix.

有一个系统范围的设置来禁用这些标志可能有点过分 - 只是序列暂停的过程可能就足够了 - 但它肯定是一个简单的修复，因为它不会影响系统的正常运行，因此它是一个安全修复。

## `PF_MEMALLOC_NOIO` in 3.9-rc1

> Just as suspend/resume has taught us that boot-time is not that much of a special case, so too runtime power management has taught us that suspend isn't all that much of a special case either. If a block device is runtime-suspended to save power, then obviously it cannot handle requests to write out a dirty page of memory until it has woken up, and until any devices it depends on (a USB controller, a PCI bus) are awake too. So none of these devices can safely perform memory allocation using `__GFP_IO`.

就像暂停/恢复告诉我们启动时间并不是特殊情况那样，运行时电源管理也告诉我们暂停并不是特殊情况。如果块设备是运行时挂起的以节省电量，那么显然它无法处理写出内存脏页的请求，直到它被唤醒，直到它依赖的任何设备（USB控制器，PCI总线）都处于唤醒状态太。因此，这些设备都不能使用__GFP_IO安全地执行内存分配。

> In order to ensure this, we could use `set_gfp_allowed_mask()` while a device was suspending or resuming, but if multiple such devices were suspending or resuming we could easily lose track of when to restore the right mask. So [this change](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=21caf2fc1931b485483ddd254b634fa8f0099963) introduces a process flag much like `PF_FSTRAN`S, only to disable `__GFP_IO` rather than `__GFP_FS`. It also takes care to record the old value whenever the flag is set, and restore that old value when done. To know when to set this flag, a `memalloc_noio` flag is introduced for each device; it is then propagated into the parents in the device tree. `PF_MEMALLOC_NOIO` is set whenever calling into the power management code for any device with `memalloc_noio` set.

为了确保这一点，我们可以在设备暂停或恢复时使用set_gfp_allowed_mask（），但如果多个此类设备暂停或恢复，我们很容易忘记何时恢复正确的掩码。因此，此更改引入了一个与PF_FSTRANS非常相似的进程标志，仅用于禁用__GFP_IO而不是__GFP_FS。每当设置标志时，它还会记录旧值，并在完成时恢复旧值。要知道何时设置此标志，将为每个设备引入memalloc_noio标志; 然后它会传播到设备树中的父节点。 PF_MEMALLOC_NOIO每当调用设置了memalloc_noio的任何设备的电源管理代码时， 都会设置。

> As both the early boot processing and the suspend/resume processing are largely single-threaded (or have dedicated threads), it is quite possible that setting `PF_MEMALLOC_NOIO` and `PF_FSTRANS` on those threads would be a sufficient alternative to using `set_gfp_allowed_mask()`. However, as there is no clear benefit from such a change, and no clear certainty that it would work, it is safer, once again, to leave that which works alone.

由于早期启动处理和挂起/恢复处理主要是单线程（或具有专用线程），因此很可能在这些线程上设置PF_MEMALLOC_NOIO和PF_FSTRANS是使用set_gfp_allowed_mask（）的充分替代方法 。但是，由于这种改变没有明显的好处，并且没有明确的确定它会起作用，所以再一次保留那些单独行动的更安全。

## Patterns that emerge

> Amid all these details there are a couple of patterns which stand out.

在所有这些细节中，有一些突出的模式。

> The first is repeated refinement of the "avoid recursion" concept. At first it was implicit in an enumerated value passed to `get_free_page()`, then it was made explicit in the first `__GFP_IO`, and then the `PF_MEMALLOC` flag. Next it was extended to cover more subtle forms of recursion with the second version of `__GFP_IO` and, finally, that was split into two separate flags to express an even wider range of recursion scenarios that can be separately avoided.

第一个是重复细化“避免递归”的概念。首先，它隐含在传递给get_free_page（）的枚举值中 ，然后在第一个__GFP_IO中显式，然后是PF_MEMALLOC标志。接下来，它扩展为使用第二版__GFP_IO覆盖更微妙的递归形式，最后，它被分成两个单独的标志，以表示可以单独避免的更广泛的递归场景。

> It is conceivable that there is room for further refinement. We could have separate flags for different sorts of locks — one for page locks and one for inode locks, for example. There is no evidence that this would presently be useful, but Linux isn't really finished yet, so we just don't know.

可以想象存在进一步改进的空间。例如，我们可以为不同类型的锁提供单独的标志 - 一个用于页锁，一个用于inode锁。没有证据表明这目前是有用的，但Linux还没有真正完成，所以我们只是不知道。

> The second pattern is the repeated discovery that just having a `GFP` flag often isn't enough — three times a new process flag was added because sometimes it isn't just a single allocation that needs to be controlled, but all allocations made by a given process. Is it only a matter of time before we get either a process flag which disables `__GFP_WAIT` or a per-process `gfp_allowed_mask`?

第二种模式是重复发现只有GFP标志通常是不够的 - 增加了三次新的进程标志，因为有时它不仅仅是需要控制的单个分配，而是由给定的所有分配处理。我们得到一个禁用__GFP_WAIT或每个进程gfp_allowed_mask的进程标志只是时间问题 吗？

> As a footnote to this pattern, it is amusing that in [3.6-rc1](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=b37f1dd0f543d9714f96c2f9b9f74f7bdfdfdf31), as part of adding support for swap-over-NFS, a new flag, `__GFP_MEMALLOC`, was added which has much the same effect as `PF_MEMALLOC` in ignoring the normal low-watermarks and providing access to the last reserves of memory. This, together with the per-socket `sk_allocation` mask, allows certain TCP sockets (those which NFS is performing swap over) to access those last reserves to make sure that swap-out always succeeds. Clearly there is need for both `GFP` flags and process flags, as well as some per-device and per-socket flags.

作为这种模式的一个注释，很有趣的是在3.6-rc1中，作为添加对swap-over-NFS的支持的一部分，添加了一个新的标志__GFP_MEMALLOC，它与PF_MEMALLOC在忽略正常的低位时有相同的效果。水印并提供对最后记忆储备的访问。这与每个套接字的sk_allocation掩码一起允许某些TCP套接字（NFS正在执行交换的那些）访问那些最后的保留，以确保交换总是成功。显然，需要GFP标志和进程标志，以及一些每设备和每插槽标志。

## We've not seen the last of this

> While studying history can be generally enlightening, it can also be specifically useful as it is in this case. Next week, we will use this understanding of memory allocation deadlocks to explore some deadlocks which have long been possible in a certain configuration, but which now need to be removed.

虽然研究历史通常具有启发性，但在这种情况下它也可以特别有用。下周，我们将使用对内存分配死锁的理解来探索在某种配置中长期存在的一些死锁，但现在需要将其删除。

[1]: http://tinylab.org
