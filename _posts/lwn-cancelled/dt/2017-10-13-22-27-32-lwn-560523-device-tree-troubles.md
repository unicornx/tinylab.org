---
layout: post
author: 'Wang Chen'
title: "LWN 文章翻译 560523 - Device tree troubles"
# tagline: " 子标题，如果存在的话 "
# album: " 所属文章系列/专辑，如果有的话"
# group: " 默认为 original，也可选 translation, news, resume or jobs, 详见 _data/groups.yml"
permalink: /lwn-560523/
description: "LWN 文章翻译 560523"
category:
  - category1
  - category2
tags:
  - tag1
  - tag2
---

> By Unicornx of [TinyLab.org][1]
> 2017-10-13 22:27:32

# 设备树的麻烦

> By Jonathan Corbet  
> July 24, 2013  

> Kernel developers working on the x86 architecture are spoiled; they develop for 
> hardware that, for the most part, identifies itself when asked, with the result 
> that it is usually easy to figure out how a specific machine is put together. 
> Other architectures — most notably ARM — are rather messier in this regard, 
> requiring the kernel to learn about the configuration of the hardware from 
> somewhere other than the hardware itself. Once upon a time, hard-coded "board files" 
> were used to build ARM-system-specific kernels; more recently, the [device tree](http://devicetree.org/Device_Tree_Usage) 
> mechanism has emerged as the preferred way to describe a system to the kernel. 
> A device tree file provides the enumeration information that the hardware itself 
> does not, allowing the kernel to understand the configuration of the system it 
> is running on. The device tree story is one of success, but, like many such stories, 
> success is bringing on some growing pains.

工作在 x86 架构上的内核开发人员都快被宠坏了; 大多数情况下，他们遇到的硬件都是支持自动识别的，所以内核很容易弄清楚一个特定的机器由什么硬件模块组成的。但是其他的体系结构 - 最着名的譬如 ARM - 在这方面的处理则相当混乱，需要人为地为内核描述硬件的配置。曾经内核开发者使用硬编码的“板级文件”来构建 ARM 系统特定的内核； 最近，[设备树](http://devicetree.org/Device_Tree_Usage)机制已经成为为内核描述系统的首选方式。一份设备树文件为那些不能自动枚举的硬件提供信息描述，从而让内核了解其当前运行的系统配置。设备树的概念提出是成功的，但是像许多类似的优秀机制一样，也面临着一些成长中所带来的问题和痛苦。

> A device tree "binding" is the specification of how a specific piece of hardware 
> can be described in the device tree data structure. Most drivers meant to run on 
> platforms where device trees are used include a documentation file describing that 
> driver's bindings; see [Documentation/devicetree/bindings/net/can/cc770.txt](https://lwn.net/Articles/560533/) as a 
> randomly chosen example. The kernel contains nearly 800 such files, plus a hundreds 
> more ".dts" files describing complete system-on-chips and boards, and the number 
> is growing rapidly.

设备树的“绑定”规则定义了如何在设备树的数据结构中描述特定硬件组件信息的规范。大多数种类的驱动程序如果要在支持设备树的平台系统上运行，则需要提供一份描述该驱动程序绑定规则的文档文件; 请参阅一个随机选择的例子[Documentation/devicetree/bindings/net/can/cc770.txt](https://lwn.net/Articles/560533/)。内核包含近 800 个这样的文件，还有数百个 “ .dts ” 文件描述了完整的片上系统和板卡，其数量正在迅速增长。

> Maintenance of those files is proving to be difficult for a number of reasons, 
> but the core of the problem can be understood by realizing that a device tree 
> binding is a sort of API that has been exposed by the kernel to the world. If a 
> driver's bindings change in an incompatible way, newer kernels may fail to boot 
> on systems with older device trees. Since the device tree is often buried in the 
> system's firmware somewhere, this kind of problem can be hard to fix. But, even 
> when the fix is easy, the kernel's normal API rules should apply; newer kernels 
> should not break on systems where older kernels work.

由于种种原因维护这些文件被证明是非常困难的，但是如果你能够意识到设备树的绑定规则也是属于内核向外部公开的一种 应用编程接口，那么这种繁重的维护工作的价值也就不难理解了。如果驱动程序的绑定规则以不兼容的方式更改，则较新的内核可能无法利用较旧的设备树启动系统。更甚的是由于设备树还经常被包含在系统固件里，这种问题就更难解决了。总而言之，无论修复是否容易，内核的通用规则必须要得到遵守; 即新内核要向后兼容旧的系统。

> The clear implication is that new device tree bindings need to be reviewed with 
> care. Any new bindings should adhere to existing conventions, they should describe 
> the hardware completely, and they should be supportable into the future. And this 
> is where the difficulties show up, in a couple of different forms: (1) most subsystem 
> maintainers are not device tree experts, and thus are not well equipped to review 
> new bindings, and (2) the maintainers who are experts in this area are overworked 
> and having a hard time keeping up.

很明显新的设备树绑定规则需要被仔细审查。任何新的定义都应该遵守现有的约定，它们应该完整地描述硬件，并且不会过快地过时。这也是比较难以处理的地方，表现为以下几种不同的形式：（1）大多数子系统的维护者并不是很精通设备树的概念，因此没有足够的能力审查新定义的绑定规则，反过来这也导致（2）那些精通设备树概念的相关专家压力巨大，花费了大量的时间审查规则是否正确。

> The first problem was the subject of [a request for a Kernel Summit discussion](https://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-July/000110.html) with 
> the goal of educating subsystem maintainers on the best practices for device tree 
> bindings. One might think that a well-written document would suffice for this purpose, 
> but, unfortunately, these best practices still seem to be in the "I know it when 
> I see it" phase of codification; as Mark Brown [put it](https://lwn.net/Articles/560544/):

第一个问题已经作为专题提请[内核峰会](https://lists.linuxfoundation.org/pipermail/ksummit-2013-discuss/2013-July/000110.html)进行讨论，目的是能够更好地对子系统的维护者展开有关设备树绑定规则的培训。人们可能会说，是不是可以提供一个完备用于判断的文档，但不幸的是，目前还无法提供这种标准文档，所以最好的做法看上去依然是根据你自己的理解去判断。正如马克·布朗在[一封邮件](https://lwn.net/Articles/560544/)中所说的那样：

> > At the minute it's about at the level of saying that if you're not sure or don't 
> > know you should get the devicetree-discuss mailing list to review it. Ideally 
> > someone would write that document, though I wouldn't hold my breath and there is 
> > a bunch of convention involved.

>  >  目前的状态是，如果你不确定或者不懂的话你最好去设备树的讨论邮件列表里去自己搜。如果有人能来写这个文档当然最好啦，但是起码我不会去写，因为这里面涉及太多的规则描述了。

> Said mailing list tends to be overflowing with driver postings, though, making it 
> less useful than one might like. Meanwhile, the best guidance, perhaps, came from 
> [David Woodhouse](https://lwn.net/Articles/560546/):

可惜邮件列表里面的信息量往往太多太杂乱，过于依赖它反而不好。也许，最好的指导来自 [David Woodhouse 的话](https://lwn.net/Articles/560546/)：

> > The biggest thing is that it should describe the *hardware*, in a fashion which 
> > is completely OS-agnostic. The same device-tree binding should work for Solaris, 
> > *BSD, Windows, eCos, and everything else.

最关键的原则是，设备数应该以完全与 OS 无关的方式描述 *硬件*。相同的设备树绑定规则应适用于 Solaris，* BSD，Windows，eCos 等所有的其他操作系统，而不能仅仅是 Linux。

> That is, evidently, not always the case, currently; some device tree bindings can 
> be strongly tied to specific kernel versions. Such bindings will be a maintenance 
> problem in the long term.

虽然只是个例，但当前的确存在一些设备数的绑定规则非常依赖于某个特定的内核版本。从长远来说，这种绑定将会带来维护问题。

> Keeping poorly-designed bindings out of the mainline is the responsibility of the 
> device tree maintainers, but, as Grant Likely (formerly one of those maintainers) 
> [put it](https://lwn.net/Articles/560547/), this maintainership "simply isn't working right now." Grant, along with 
> Rob Herring, is unable to keep up with the stream of new bindings (over 100 of 
> which appeared in 3.11), so a lot of substandard bindings are finding their way 
> in. To address this problem, Grant has announced a "refactoring" of how device 
> tree maintainership works.

将设计不力的绑定规则排除在主线之外是设备树维护者的责任，但是像 Grant Likely（前维护者之一）[所说的那样](https://lwn.net/Articles/560547/)，这么做 "根本行不通" 。Grant 和 Rob Herring 根本应付不了如洪水般新出现的绑定规则定义（光 3.11 版本的内核中就有超过100个新的定义），所以很多不合标准的定义没有经过严格的审查就被合入了主线。为了解决这个问题，Grant 提出要对设备树相关的维护工作进行“重建”。

> The first part of that refactoring is Grant's own resignation, with lack of time 
> given as the reason. In his place, four new maintainers (Pawel Moll, Mark Rutland, 
> Stephen Warren and Ian Campbell) have been named as being willing to join Rob and 
> take responsibility for device tree bindings; others with an interest in this area 
> are encouraged to join this group.

这个“重建”工作的第一部分是 Grant 本人不再担任设备树的维护工作，由于种种时间上的原因。为了替代他的位置，四名新的维护者（Pawel Moll，Mark Rutland，Stephen Warren 和 Ian Campbell）加入和 Rob 一起负责设备树绑定规则的维护; 同时也欢迎其他对此有兴趣的人加入这个小组。

> The next step will be for this group to figure out how device tree maintenance will 
> actually work; as Grant noted, "There is not yet any process for binding maintainership." 
> For example, should there be a separate repository for device tree bindings (which 
> would make review easier), or should they continue to be merged through the relevant 
> subsystem trees (keeping the code and the bindings together)? It will take some 
> time, and possibly a Kernel Summit discussion, to figure out a proper mechanism 
> for the sustainable maintenance of device tree bindings.

队伍建立起来后，下一步就是确定设备树维护工作的方式; Grant 指出，“ 需要建立相应的流程 ”。例如，是否可以为设备树绑定规则单独新建一个代码库（这将使审核更容易），还是或者依然通过相关的子系统的 源码树来合并（这么做代码和绑定规则仍然在一起提交）？这需要一些时间，也可能需要通过内核峰会上的讨论，以确定一个适当的机制来实现设备树绑定的可持续维护。

> Some other changes are in the works. The kernel currently contains hundreds of .dts 
> files providing complete device trees for specific systems; there are also many .dtsi 
> files describing subsystems that can be included into a complete device tree. In 
> the short term, there are plans to design a schema that can be used to formally 
> describe device tree bindings; the device tree compiler utility (dtc) will then 
> be able to verify that a given device tree file adheres to the schema. In the 
> longer term, those device tree files are likely to move out of the kernel entirely 
> (though the binding documentation for specific devices will almost certainly remain).

其他一些改变正在进行中。内核目前包含数百个 `.dts` 文件，这些文件为特定系统提供完整的设备树描述; 还有许多描述子系统的 `.dtsi` 文件可以被包含在一个完整的设备树中。短期内，有计划设计一套可用于正式描述设备树绑定的模版规则; 这样设备树的编译器（dtc）将能够根据这套规则验证给定的设备树文件格式。从长远来看，这些设备树文件可能会被完全移出内核（但特定设备的绑定规则文档肯定会保留）。

> All told, the difficulties with device trees do not appear to be anything other 
> than normal growing pains. A facility that was once only used for a handful of 
> PowerPC machines (in the Linux context, anyway) is rapidly expanding to cover a 
> sprawling architecture that is in wide use. Some challenges are to be expected 
> in a situation like that. With luck and a fair amount of work, a better set of 
> processes and guidelines for device tree bindings will result from the 
> discussion — eventually.

总而言之，设备树的困难并不是无法克服的。曾经只用于少量 PowerPC 机器的设备树机制（我指的是针对 Linux环境下）正在迅速扩大到其他的体系架构。在这种情况下可以预见到一些挑战。相信通过努力，最终会产生一套更好的流程和指导规则来定义设备树绑定规则。
