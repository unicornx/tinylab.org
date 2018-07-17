---
layout: post
author: 'Wang Chen'
title: "LWN 414016: Grant Likely 有关设备树的报道"
album: 'LWN 中文翻译'
group: translation
permalink: /lwn-414016-grant-likely-on-device-trees/
description: "LWN 文章翻译，Grant Likely 有关设备树的报道"
category:
  - 设备树
  - LWN
tags:
  - Linux
  - device tree
---

> 原文：[ELCE: Grant Likely on device trees](https://lwn.net/Articles/414016/)
> 原创：By By Jake Edge @ November 10, 2010
> 翻译：By Unicornx of [TinyLab.org][1]
> 校对：By 

> Device trees are a fairly hot topic in the embedded Linux world as a means to more easily support multiple system-on-chip (SoC) devices with a single kernel image. Much of the work implementing device trees for the PowerPC architecture, as well as making that code more generic so that others could use it, has been done by Grant Likely. He spoke at the recent Embedded Linux Conference Europe (ELCE) to explain what device trees are, what they can do, and to update the attendees on efforts to allow the ARM architecture use them.

在嵌入式 Linux 的世界里设备树是一个相当热门的话题，因为采用该技术可以实现用一个内核镜像支持多款 SoC (译者注：system-on-chip 片上系统) 设备。Grant Likely 是 PowerPC 架构上支持设备树的主要实现者并且实现了相关代码的通用化。在最近的欧洲嵌入式 Linux 大会 (Embedded Linux Conference Europe 简称 ELCE) 上他为与会者介绍了设备树的概念和可以利用设备树实现的功能以便 ARM 体系架构的开发人员也可以尽快使用该技术。

> ![](https://static.lwn.net/images/2010/elce-likely-sm.jpg) All of the work that is going into adding device tree support for various architectures is not being done for an immediate benefit to users, Likely said. It is, instead, being done to make it easier to manage embedded Linux distributions, while simplifying the boot process. It will also make it easier to port devices (i.e. components and "IP blocks") to different SoCs. But it is "not going to make your Android phone faster".

按照 Likely 的说法，增加设备树对各种不同体系架构的支持对于使用者来说并不会产生立即的好处。这么做的好处在于简化引导启动 (boot) 流程从而使对嵌入式 Linux 发布包的管理更容易。同时设备树也减少了设备（即组件或者称之为 IP 模块，译者注，这里的 IP 指的是 Intellectual Property）在各种片上系统中的的移植工作量。但这些改进 "并不会使你的安卓手机更快" ( Likely 的原话 "not going to make your Android phone faster")。

> A device tree is just a data structure that came from OpenFirmware. It represents the devices that are part of particular system, such that it can be passed to the kernel at boot time, and the kernel can initialize and use those devices. For architectures that don't use device trees, C code must be written to add all of the different devices that are present in the hardware. Unlike desktop and server systems, many embedded SoCs do not provide a way to enumerate their devices at boot time. That means developers have to hardcode the devices, their addresses, interrupts, and so on, into the kernel.

设备树的概念来自 OpenFirmware 定义的一种数据结构。可以用来表示一个特定系统的硬件组成，该数据结构信息在系统启动阶段被传递给内核读取，这样内核就可以利用这些信息来初始化和使用这些硬件设备。对于哪些没有使用设备树的体系架构，需要采用 C 的方式对硬件信息进行硬编码。和桌面系统以及服务器系统不同，许多嵌入式的片上系统并没有提供一种在系统引导阶段枚举识别其片上设备的能力。这意味着开发人员不得不采用硬编码的方式在内核中描述这些设备，包括告诉内核这些设诶的地址，中断线等等。

> The requirement to put all of the device definitions into C code is hard to manage, Likely said. Each different SoC variant has to have its own, slightly tweaked kernel version. In addition, the full configuration of the device is scattered over multiple C files, rather than kept in a single place. Device trees can change all of that.

采用硬编码的方式来给出设备的定义非常难于管理，Likely 说。每一种不同的片上系统有对应其自己特殊修改的内核版本。此外，完整的设备配置分散在多个 C 源文件中，而不是集中在一个地方。设备树可以改进这些问题。

> A device tree consists of a set of nodes with properties, which are simple key-value pairs. The nodes are organized into a tree structure, unsurprisingly, and the property values can store arbitrary data types. In addition, there are some standard usage conventions for properties so that they can be reused in various ways. The most important of these is the compatible property that uniquely defines devices, but there are also conventions for specifying address ranges, IRQs, GPIOs, and so forth.

一个设备树由很多节点构成，每个节点包含自己的属性，这些属性的形式是简单的键-值对 (译者注：key-value pair)。所有的节点被组织成一个树状结构，属性值可以表示任意的数据类型。此外，对于属性有一些标准的使用方式便于其在各种条件下的复用。在这些属性中最重要的是键名为 `compatible` 的属性，用于唯一地定义设备，此外还有一些固定的表示方法用于表示地址范围，中断 (IRQ)，通用输入输出管脚(GPIO) 等等。
 
> Likely used a simplified example from devicetree.org to show what these trees look like. They are defined with an essentially C-like syntax:

Likely 使用了一个来自 `devicetree.org` 的简化版本的例子来延时设备树的形式，其描述形式看上去很像 C 代码的编写风格。

	/ {
		compatible = "acme,coyotes-revenge";
	
		cpus {
			cpu@0 {
				compatible = "arm,cortex-a9";
			};
			cpu@1 {
				compatible = "arm,cortex-a9";
			};
		};

		serial@101F0000 {
			compatible = "arm,pl011";
		};
		...
		external-bus {
			ethernet@0,0 {
				compatible = "smc,smc91c111";
			};
		
			i2c@1,0 {
				compatible = "acme,a1234-i2c-bus";
				rtc@58 {
					compatible = "maxim,ds1338";
				};
			};
			...

> The compatible tags allow companies to define their own namespace ("acme", "arm", "smc", and "maxim" in the example) that they can manage however they like. The kernel already knows how to attach an ethernet device to a local bus or a temperature sensor to an i2c bus, so why redo it in C for every different SoC, he asked. By parsing the device tree (or the binary "flattened" device tree), the kernel can set up the device bindings that it finds in the tree.

`compatible` 属性允许各个公司定义自己的名字空间(见例子中的 "acme", "arm", "smc", and "maxim")。内核已经具备将一个以太网络设备连接到本地的总线上或者将一个温度传感器连接到一个 I2C 总线上的能力，所以我们没有必要为每个不同的片上系统再重新写一遍类似的代码了。通过解析设备树 (实际运行时是通过解析二进制的扁平化 "flattened" 的设备树)，内核可以建立设备和驱动的绑定。

> One of the questions that he often gets asked is: "why bother changing what we already have?" That is a "hard question to answer" in some ways, because for a lot of situations, what we have in the kernel currently does work. But in order to support large numbers of SoCs with a single kernel (or perhaps a small set of kernels), something like device tree is required. Both Google (for Android) and Canonical (for Linaro) are very interested in seeing device tree support for ARM.

Likely 说他经常被问及的问题是："现在的工作方式好好的为何你还要费那么大的劲再引入设备树这个新的概念？" 这的确是一个很难回答的问题，因为在很多情况下，当前的处理方式也是可以运行的。但为了支持使用单个或少量内核版本同时支持多种片上系统，我们就需要引入设备树了。譬如 Google 在 Android 项目上，以及 Canonical 在 Linaro 项目上都非常希望使用设备树来支持基于 ARM 的产品。

> Beyond that, "going data-driven to describe our platforms is the right thing to do". There is proof that it works in the x86 world as "that's how it's been done for a long time". PowerPC converted to device trees five years ago or so and it works well. There may be architectures that won't need to support multiple devices with a single kernel, and device trees may not be the right choice for those, but for most of the architectures that Linux supports, Likely clearly thinks that device trees are the right solution.

除此之外，按照 Likely 的原话 “采用基于数据驱动的方式来描述我们的平台是一种正确的工作方式”("going data-driven to describe our platforms is the right thing to do")。证据表明，在 x86 的世界里 “这么做已经有很长一段时间”。 PowerPC 在五年前开始转而采用设备树，目前运行良好。当然存在一些体系架构并不希望采用单个内核来支持多款产品，那么这件事是不需要的，但对 Linux 支持的大部分体系架构来说，设备树一定是更好的选择。

> He next looked at what device trees aren't. They don't replace board-specific code, and developers will "still have to write drivers for weird stuff". Instead, device trees simplify the common case. Device tree is also not a boot architecture, it's "just a data structure". Ideally, the firmware will pass a device tree to the kernel at boot time, but it doesn't have to be done that way. The device tree could be included into the kernel image. There are plenty of devices with firmware that doesn't know about device trees, Likely said, and they won't have to.

他接着阐述了设备树并不是万能的。它并没有完全代替板级相关的代码，开发人员依然“需要为自己特定的功能编写自己的驱动代码”。但采用设备树可以简化一些通用的处理。设备树也不是一个启动引导的架构，“它只是一个数据结构”。通常情况下，固件会在启动引导阶段将设备树传递给内核，但这也不是唯一的方法。我们可以将设备树信息打包在内核镜像中。目前还存在很多设备它们的固件并不支持设备树，Likely 说。

> There is currently a push to get ARM devices into servers, as they can provide lots of cores at low power usage. In order to facilitate that, there needs to be one CD that can boot any of those servers, like it is in the x86 world. Device trees are what will be used to make that happen, Likely said.

因为 ARM 处理器支持许多核心以及低功耗方面的优点，当前有很大的需求要将 ARM 处理器应用到服务器产品上去。为此，我们需要实现一张光盘启动各种型号的服务器，就像我们现在在 x86 处理器上一样。那么设备树将变得非常有必要。

> Firmware that does support device trees will obtain a .dtb (i.e. flattened device tree binary) file from somewhere in memory, and either pass it verbatim to the kernel or modify it before passing. Another option would be for the firmware to create the .dtb on-the-fly, which is what OpenFirmware does, but that is a "dangerous" option. It is much easier to change the kernel than the firmware, so any bugs in the firmware's .dtb creation code will inevitably be worked around in the kernel. In any case, the kernel doesn't care how the .dtb is created.

支持设备树的固件将从存储设备中读取一个 `.dtb` 文件，这个文件是二进制格式的，包含了我们前述的设备树的描述信息，固件将该文件内容原封不动地传递给内核或者对其略加修改后再传递给内核。还有一种方式，也是 OpenFirmware 采用的方式是在运行过程中由固件自己创建该设备树信息，但这么做存在一定的风险。相对来说替换内核比替换固件要方便得多，一旦固件中创建设备树的代码存在问题则需要内核来做相应的修补。总而言之，内核不关心设备树是如何创建的。（译者注，从原文字面上感觉作者想表达的意思是这种让固件动态生成设备树的方式是不推荐的。）

> For ARM, the plan is to pass a device tree, rather than the existing, rather inflexible ARM device configuration known as ATAGs. The kernel will set up the memory for the processor and unflatten the .dtb into memory. It will unpack it into a "live tree" that can then be directly dereferenced and used by the kernel to register devices.

对于 ARM 体系架构来说，计划是用设备树来代替现存的 ATAG 方式传递硬件描述信息。内核将在内存中将二进制的设备树信息解析展开成树状结构，然后基于这棵设备树来引用和注册设备。

> The Linux device model is also tree-based, and there is some congruence between device tree and the device model, but there is not a direct 1-to-1 mapping between them. That was done "quite deliberately" as the design goal was "not to describe what Linux wants", instead it was meant to describe the hardware. Over time, the Linux device model will change, so hardcoding Linux-specific values into the device tree has been avoided. The device tree is meant to be used as support data, and the devices it describes get registered using the Linux device model.

Linux 的设备模型也是基于树状的结构，设备模型和设备树在某些信息上是重叠的，但绝不是完全一一对应的。设备树在设计上的一个目标就是专注于在硬件层面描述设备信息，而不是像设备模型专注于描述设备驱动总线绑定的关系等内核所关心的东西。随着时间的推移和内核的发展，设备模型会随之变化，但设备树所描述的信息应该尽可能和内核无关。设备树仅仅用于描述硬件数据，Linux 基于这些信息在内部转化为内核设备对象并将这些设备对象注册到设备模型上去。

> Device drivers will match compatible property values with device nodes in a device tree. It is the driver that will determine how to configure the device based on its description in a device tree. None of that configuration code lives in the device tree handling, it is part of the drivers which can then be built as loadable kernel modules.

内核扫描设备树中的设备节点，并根据节点的 `compatible` 属性来匹配设备驱动。驱动被加载后继续基于设备树的描述来执行具体配置硬件设备的动作。内核中存在通用的设备树处理逻辑但不会对具体设备进行具体的配置处理，具体的配置动作由绑定的驱动代码完成，而驱动可以被编译为可动态加载的内核模块。

> Over the last year, Likely has spent a lot of time making the device tree support be generic. Previously, there were three separate copies of much of the support code (for Microblaze, SPARC, and PowerPC). He has removed any endian dependencies so that any architecture can use device trees. Most of that work is now done and in the mainline. There is some minimal board support that has not yet been mainlined. The MIPS architecture has added device tree support as of 2.6.37-rc1 and x86 was close to getting it for 2.6.37, but some last minute changes caused the x86 device tree support to be held back until 2.6.38.

在过去的一年里， Likely 花费了大量的时间改进内核设备树的通用化支持。从前对设备树的支持不同的平台代码是不可复用的（譬如 Microblaze, SPARC, 和 PowerPC）。他移除了代码中对大小端的依赖性实现了代码对不同体系架构的通用性。相关工作大部分已经完成并且进入了内核主线，除了少量板级支持的代码。MIPS 架构从 2.6.37-rc1 开始增加可设备树支持，x86 在 2.6.37 时已经接近完成但由于一些变故导致内核对 x86 的支持延迟到 2.6.38。

> The ARM architecture still doesn't have device tree support and ARM maintainer Russell King is "nervous about merging an unmaintainable mess". King is taking a wait-and-see approach until a real ARM board has device tree support. Likely agreed with that approach and ELCE provided an opportunity for him and King to sit down and discuss the issue. In the next six months or so (2.6.39 or 2.6.40), Likely expects that the board support will be completed and he seems confident that ARM device tree support in the mainline won't be far behind.

ARM 架构还未支持设备树，其维护负责人 Russell King 表示“过快的加入该功能会导致无法维护的混乱”。所以他倾向于采用观望的态度，等到有实际的 ARM 产品尝试完成设备树的支持。 Likely 表示不反对他这么做并且他认为 ELCE 提供了这个机会让他和 King 能够坐到一起开讨论这个问题。在接下来的六个月左右（估计到 2.6.39 或者 2.6.40）时 Likely 期望基于 ARM 的板级支持可以完成并且他坚信内核主线对 ARM 体系的设备树支持不会被拉下太多。

> There are other tasks to complete in addition to the board support, of course, with documentation being high on that list. There is a need for documentation on how to use device trees, and on the property conventions that are being used. The devicetree.org wiki is a gathering point for much of that work.

当然，还有很多工作需要万册好难过，譬如文档的完善。社区需要详细的文档介绍如何使用设备树，如何解析属性。当前所有的这些信息集中在 devicetree.org 的 wiki 上进行收集整理。

> There were several audience questions that Likely addressed, including the suitability of device tree for Video4Linux (very suitable and the compatible property gives each device manufacturer its own namespace), the performance impact (no complaints, though he hasn't profiled it — device trees are typically 4-8K in size, which should minimize their impact), and licensing or patent issues (none known so far, the code is under a BSD license so it can be used by proprietary vendors — IBM's lawyers don't seem concerned). Overall, both Likely and the audience seemed very optimistic about the future for device trees in general and specifically for their future application in the ARM architecture.

现场 Likely 回答了一些观众的问题，包括设备树是否适用于 Video4Linux (答案是没有问题，`compatible` 属性赋予设备制造商足够的名字空间)，设备树是否存在性能问题 (答案是没有问题，虽然 Likely 没有给出具体的数据，典型情况下一个设备树的大小是 4 到 8K)，使用设备树是否存在许可证和专利问题(回答是目前位置没有看到有任何问题，设备树的代码遵行 BSD 许可证所以可以被专有厂商所使用 - IBM 的律师看上去不关心这个)。总之，Likely 和与会的听众对设备树的通用化和未来在 ARM 架构上的应用是非常乐观的。