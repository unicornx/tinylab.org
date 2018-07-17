---
layout: post
author: 'Wang Chen'
title: "LWN 文章翻译 573409 - Device trees II: The harder parts"
# tagline: " 子标题，如果存在的话 "
# album: " 所属文章系列/专辑，如果有的话"
# group: " 默认为 original，也可选 translation, news, resume or jobs, 详见 _data/groups.yml"
permalink: /lwn-573409/
description: "LWN 文章翻译 573409"
category:
  - category1
  - category2
tags:
  - tag1
  - tag2
---

> By Unicornx of [TinyLab.org][1]
> 2017-10-13 22:27:32

# 设备树绑定 (Device tree bindings)

> By Jonathan Corbet  
> July 30, 2013  
> [2013 Kernel Summit](https://lwn.net/Articles/KernelSummit2013/)
 
> The device tree issue was omnipresent during the 2013 Kernel Summit, with dedicated 
> minisummit sessions, hallway discussions, and an interminable mailing list thread 
> all in the mix. Despite all the noise, though, some progress was seemingly made 
> on the issue of how to evolve device tree bindings without breaking systems that 
> depend on them. A group of developers presented those results to the plenary session.

有关设备树问题的讨论在 2013 年内核峰会期间无处不在，专门的小型会议上，走廊讨论中以及无数的邮件列表都议论这一话题。尽管如此，在如何进一步开展设备树绑定开发而不破坏依赖于其的系统这个问题上，似乎已经取得了一些进展。一组开发人员将这些结果提交给了全体会议。

> Grant Likely and David Woodhouse started by reiterating the problems that led to 
> the adoption of device trees for the ARM architecture in the first place. It comes 
> down to undiscoverable hardware — hardware that does not describe itself to the 
> CPU, and which thus cannot be enumerated automatically. This hardware is 
> ![David Woodhouse](https://static.lwn.net/images/conf/2013/lce-ks/DavidWoodhouse-sm.jpg) 
> not just a problem with embedded ARM systems, Grant said; it is showing up in 
> desktop and server systems too. In many situations, we are seeing the need for a 
> general hardware description mechanism. The problem is coming up with the best 
> way of doing this description while supporting systems that were shipped with 
> intermediate device tree versions.

Grant Likely 和 David Woodhouse 首先重点总结了促使 ARM 架构尽快采用设备树的问题。关键问题在于那些不可被处理器自动发现的硬件。![David Woodhouse](https://static.lwn.net/images/conf/2013/lce-ks/DavidWoodhouse-sm.jpg)提出这些硬件不仅仅是嵌入式 ARM 系统的一个问题，Grant 说：该问题也会出现在桌面系统和服务器系统中。在许多情况下，我们需要一个通用的硬件描述机制。最终的问题被描述为我们需要发明一种最佳的描述方法可以支持那些采用中间版本的设备树描述发布产品的系统。

> The solution comes down to a set of process changes, starting with a statement 
> that device tree bindings are, indeed, considered to be stable by default. Once 
> a binding has been included in a kernel release, developers should not break 
> systems that are using that binding. That said, developers should not get hung 
> up on creating perfect bindings now; we still do not know all of the common 
> patterns and will need to make changes as we learn things. That means that 
> bindings can, in fact, change after they have been released in a kernel; the key 
> is to make those changes in the correct way.

解决方案归结为一组流程更改，从设备树绑定规则的声明开始，主线内核中的定义在默认情况下需要被大家按照稳定版本支持。一旦绑定随某一个内核版本发布，开发人员在使用该版本内核的系统上需要严格遵守该绑定的定义。也就是说，开发人员现在不应该随意忽略其定义而使用自己私有的绑定定义; 我们当前还不能给出所有通用的的模式，随着大家理解的深入还会继续修改。这意味着绑定规则事实上仍然可以在新内核发布中进行改变; 关键是我们需要按照正确的方式进行这些更改。

> Another decision that has been made is that configuration data is allowed within 
> device tree bindings. This has been a controversial area; many developers feel 
> that device trees should describe the hardware and nothing else. Grant made the 
> claim that much configuration data should be considered part of the hardware 
> design; there may be a region of memory intended for use as graphics buffers, 
> for example.

另一个决定是在设备树绑定规则中允许对配置数据进行描述。这曾经是一个有争议的话题； 许多开发人员认为设备树应该仅仅描述硬件。Grant 声称许多配置数据应被视为硬件设计的一部分; 例如，可能存在需要配置一段存储器区域用于图形缓存。

> There will be a staging-like mechanism for unstable bindings, but it expected 
> that this mechanism will almost never be used. The device tree developers will 
> be producing a document describing the recommended best practices and processes 
> around device trees; there will also be a set of validation tools. Much of this 
> work, it is hoped, will be completed within the next year.

我们将定义一个过渡机制来处理那些目前还不稳定的绑定规则定义，但是估计这种机制几乎永远不会被使用。设备树的开发人员将编写一个文档用于推荐处理设备树的最佳实践和过程; 我们还将提供一套验证工具。希望大部分的工作能够在明年内完成。

> The current rule that device tree bindings must be documented will be reinforced. 
> The documentation lives in Documentation/devicetree/bindings in the kernel tree. 
> The device tree maintainers would prefer to see these documents posted as a separate 
> patch within a series so they can find it quickly. Bindings should get an acknowledgment 
> from the device tree maintainers, but there is already too much review work to be 
> done in this area. So, if the device tree maintainers are slow in getting to a patch, 
> subsystem maintainers are considered empowered to merge bindings without an ack. 
> These changes should go through the usual subsystem tree.

当前已经规定设备树的绑定规则需要用文档正式记录下来，该要求需要进一步强调。所有相关文档目前放在内核源码树的 `Documentation/devicetree/bindings` 目录下。为了方便设备树维护者快速查找，建议提交者在提交文档修改时用一个单独的补丁来提交。
新的绑定规则定义要进入主线内核应该得到设备树维护人员的确认，但在这方面有太多的审查工作要做。因此，如果设备树维护者的反应比较慢的话，子系统维护者将有权决定是否合入新改动，而无需等待设备树维护人员的确认。这些改动应该通过通常的子系统树修改来完成（译者注，这句话不是很明白原作者是在说什么，主要是译者不太清楚子系统树的概念，可能和内核的开发流程相关）。

> The compatibility rules say that new kernels must work with older device trees. 
> If changes are required, they should be put into new properties; the old ones can 
> then be deprecated but not removed. New properties should be optional, so that 
> device trees lacking those properties continue to work. The device tree developers 
> will provide a set of guidelines for the creation of future-proof bindings.

内核的兼容性规则要求新内核必须支持旧的设备树。如果要对已经发布的设备树绑定规则进行更改，则应将新的改动作为新属性添加; 旧的属性可以被逐渐弃用但不能被直接删除。新属性应该是可选的，从而保证旧的缺少这些属性的设备树能够继续工作。设备树开发人员需要提供帮助说明来指导其他人创建新的绑定规则。

> If it becomes absolutely necessary to introduce an incompatible change, Grant said, 
> the first step is that the developer must submit to the mandatory public flogging. 
> After that, if need be, developers should come up with a new "compatible" string 
> and start over, while, of course, still binding against the older string if that 
> is all that is available. DTS files (which hold a complete device tree for a 
> specific system) should contain either the new or the old compatible string, 
> but never both.

如果真的有必要引入一个不兼容的改动，Grant 说，那么首先开发人员必须要将其提供给设备树维护人员以及社区并邀请大家进行仔细的审核。之后，其次，如果的确有所需要，开发人员应该定义一个新的 "compatible" 属性字符串，以便和旧的 "compatible" 属性字段相区分，旧的驱动依然可以根据旧的字段完成绑定。DTS文件（其中包含特定系统的完整设备树）可以包含新的或旧的 "compatible" 字符串，但不能同时都包含。

> If all else fails, it is still permissible to add quirks in the code for specific 
> hardware. If this is done with care, it should not reproduce the old board file 
> problem; such quirks should be relatively rare.

如果以上方法都行不通，内核仍然允许在为了一些特殊的硬件在代码中添加特殊处理。如果我们谨慎小心地对待这类异常情况，就不会重蹈过去板级文件的问题; 更何况此类特殊处理应该是很少见的。

> ![Ben Herrenschmidt](https://static.lwn.net/images/conf/2013/lce-ks/BenHerrenschmidt-sm.jpg) 
> Ben Herrenschmidt worried about the unstable binding mechanism; 
> it is inevitable, he thought, that manufacturers would ship hardware using unstable 
> bindings. David replied that bad manufacturer behavior is not limited to bindings; 
> they ship a lot of strange code as well. But, he said, manufacturers have learned 
> over time that things go a lot easier if they work with upstream-supported code. 
> He didn't think that the unstable binding mechanism would ever be used; it is a 
> "political compromise" that should never need to be employed. Arnd Bergmann added 
> that, should this ever happen, it will not be the end of the world; the kernel 
> community just has to make the consequences of shipping unstable bindings clear. 
> In such cases, users will just have to update the device tree in their hardware 
> before they can install a newer kernel.

![Ben Herrenschmidt](https://static.lwn.net/images/conf/2013/lce-ks/BenHerrenschmidt-sm.jpg) Ben Herrenschmidt 仍然对目前存在不稳定的绑定规则接口表示担忧; 他认为制造商使用不稳定的绑定机制来发布产品这种行为是不可避免的。David 答复说，制造商那些糟糕的行为其实不仅仅限于设备树绑定这一件事情上; 他们在发布的产品中添加了很多奇奇怪怪的代码。但是，他说，随着时间的推移制造商终归会理解到，如果他们使用内核主线支持的稳定代码，事情会变得更加容易。他认为人们不会永远使用那些不稳定的绑定规则; 在这个问题上内核社区绝不应该“妥协”将就。Arnd Bergmann 补充说，如果这类事情真的发生，也不是无法挽回; 内核社区只需要在新的内核版本发布中将那些不稳定的绑定规则变得明确。而用户要做的，仅仅是在升级新的内核之前需要先更新一下其硬件中的设备树。

> What about the reviewer bandwidth problem? The main change in this area, it seems, 
> is that the device tree reviewers will only look at the binding documentation; they 
> will not look at the driver code itself. That is part of why they want the documentation 
> in a separate patch. That means that subsystem maintainers will have to be a bit 
> more involved in ensuring that the code matches the documentation — though there 
> will be some tools that will help in that area as well.

那么如何解决审核小组人手不足的问题呢？在这点上的目前的流程改变是：设备树审查者只关心绑定规则文档; 他们不会去检查具体实现绑定的驱动程序代码本身。这就是为什么他们希望对绑定规则文档的修改以单独的补丁方式提交的原因。这也意味着子系统的维护人员必须更多地参与确保驱动代码与规则描述文档匹配 - 尽管社区也会为此开发一些辅助工具。

