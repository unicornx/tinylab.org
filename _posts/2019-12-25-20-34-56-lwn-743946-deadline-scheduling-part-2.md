---
layout: post
draft: true
author: 'Wang Chen'
title: "LWN 743946: Deadline 调度介绍的第二部分：xxx"
album: 'LWN 中文翻译'
group: translation
license: "cc-by-sa-4.0"
permalink: /lwn-743946/
description: "LWN 中文翻译，Deadline 调度介绍的第二部分：xxxx"
category:
  - 进程调度
  - LWN
tags:
  - Linux
  - scheduling
  - deadline
---

**了解更多有关 “LWN 中文翻译计划”，请点击 [这里](/lwn/)**

> 原文：[Deadline scheduler part 2 — details and usage](https://lwn.net/Articles/743946/)
> 原创：By Daniel Bristot de Oliveira @ Jan. 19, 2018
> 翻译：By [unicornx](https://github.com/unicornx)
> 校对：By [xxx](https://github.com/xxx)

> Linux’s deadline scheduler is a global early deadline first scheduler for sporadic tasks with constrained deadlines. These terms were defined in [the first part of this series](https://lwn.net/Articles/743740/). In this installment, the details of the Linux deadline scheduler and how it can be used will be examined.

Linux的截止时间调度程序是全球性的早期截止时间优先调度程序，用于期限有限的零星任务。这些术语在本系列的第一部分中进行了定义。在本期中，将研究Linux截止日期调度程序的详细信息以及如何使用它。

> The deadline scheduler prioritizes the tasks according to the task’s job deadline: the earliest absolute deadline first. For a system with ***M*** processors, the ***M*** earliest deadline jobs will be selected to run on the ***M*** processors.

截止时间调度程序根据任务的工作截止时间对任务进行优先级排序：最早的绝对截止时间优先。对于具有M个 处理器的系统，将选择M个最早的截止时间作业以在M个 处理器上运行。

> The Linux deadline scheduler also implements the constant bandwidth server (CBS) algorithm, which is a resource-reservation protocol. CBS is used to guarantee that each task will receive its full run time during every period. At every activation of a task, the CBS replenishes the task’s run time. As the job runs, it consumes that time; if the task runs out, it will be throttled and descheduled. In this case, the task will be able to run only after the next replenishment at the beginning of the next period. Therefore, CBS is used to both guarantee each task’s CPU time based on its timing requirements and to prevent a misbehaving task from running for more than its run time and causing problems to other jobs.

Linux截止期限调度程序还实现了恒定带宽服务器（CBS）算法，这是一种资源保留协议。CBS用于确保每个任务在每个周期都将收到其完整的运行时间。每次激活任务时，CBS都会补充任务的运行时间。随着工作的进行，它会浪费时间。如果任务用完，它将被限制和调度。在这种情况下，该任务将只能在下一个周期开始的下一次补充之后运行。因此，CBS既可以根据每个任务的时间要求来保证每个任务的CPU时间，又可以防止行为异常的任务运行超过其运行时间，并且不会给其他作业造成问题。

> In order to avoid overloading the system with deadline tasks, the deadline scheduler implements an acceptance test, which is done every time a task is configured to run with the deadline scheduler. This test guarantees that deadline tasks will not use more than the maximum amount of the system's CPU time, which is specified using the `kernel.sched_rt_runtime_us` and `kernel.sched_rt_period_us` sysctl knobs. The default values are 950000 and 1000000, respectively, limiting realtime tasks to 950,000µs of CPU time every 1s of wall-clock time. For a single-core system, this test is both necessary and sufficient. It means that the acceptance of a task guarantees that the task will be able to use all the run time allocated to it before its deadline.

为了避免使用截止期限任务使系统过载，截止期限调度程序将执行验收测试，每次将任务配置为使用截止期限调度程序运行时，都会进行一次接受测试。此测试可确保截止日期任务使用的系统CPU时间不会超过系统的最大时间，该时间是使用kernel.sched_rt_runtime_us和 kernel.sched_rt_period_us sysctl旋钮指定的。默认值分别为950000和1000000，这将实时任务限制为每1秒钟的挂钟时间950,000µs的CPU时间。对于单核系统，此测试既必要又充分。这意味着任务的接受保证了该任务将能够使用在其截止日期之前分配给它的所有运行时间。

> However, it is worth noting that this acceptance test is necessary, but not sufficient, for global scheduling on multiprocessor systems. As Dhall’s effect (described in the first part of this series) shows, the global deadline scheduler acceptance task is unable to schedule the task set even though there is CPU time available. Hence, the current acceptance test does not guarantee that, once accepted, the tasks will be able to use all the assigned run time before their deadlines. The best the current acceptance task can guarantee is bounded tardiness, which is a good guarantee for soft real-time systems. If the user wants to guarantee that all tasks will meet their deadlines, the user has to either use a partitioned approach or to use a necessary and sufficient acceptance test, defined by:

但是，值得注意的是，对于多处理器系统上的全局调度，此验收测试是必要的，但还不够。正如Dhall的效果（在本系列的第一部分中所述）所示，即使有可用的CPU时间，全局截止时间调度程序接受任务也无法调度任务集。因此，当前的验收测试不能保证一旦被接受，任务将能够在截止日期之前使用所有分配的运行时间。当前接受任务所能保证的最好的限制是拖延时间，这对于软实时系统是一个很好的保证。如果用户希望保证所有任务都将按时完成，则用户必须使用分区方法或使用必要和足够的验收测试，定义如下：

Σ(WCETi / Pi) <= M - (M - 1) x Umax

> Or, expressed in words: the sum of the run time/period of each task should be less than or equal to the number of processors, minus the largest utilization multiplied by the number of processors minus one. It turns out that, the bigger Umax is, the less load the system is able to handle.

或者用字表示：每个任务的运行时间/期间之和应小于或等于处理器数量，减去最大利用率乘以处理器数量再减去一。事实证明，Umax越大，系统处理的负载越少。

> In the presence of tasks with a big utilization, one good strategy is to partition the system and isolate some high-load tasks in a way that allows the small-utilization tasks to be globally scheduled on a different set of CPUs. Currently, the deadline scheduler does not enable the user to set the affinity of a thread, but it is possible to partition a system using control-group cpusets.

在存在利用率高的任务的情况下，一个好的策略是对系统进行分区并隔离一些高负载的任务，这种方式应允许将利用率低的任务全局调度在不同的CPU组上。当前，最后期限调度程序无法使用户设置线程的亲和力，但是可以使用控制组cpuset对系统进行分区。

> For example, consider a system with eight CPUs. One big task has a utilization close to 90% of one CPU, while a set of many other tasks have a lower utilization. In this environment, one recommended setup would be to isolate CPU0 to run the high-utilization task while allowing the other tasks to run in the remaining CPUs. To configure this environment, the user must follow the following steps:

例如，考虑具有八个CPU的系统。一个大任务的利用率接近一个CPU的90％，而其他许多任务中的一组则利用率较低。在这种环境下，建议的设置是隔离CPU0以运行高利用率任务，同时允许其他任务在其余CPU中运行。要配置此环境，用户必须遵循以下步骤：

> 1. Enter in the cpuset directory and create two cpusets:

1. 输入cpuset目录并创建两个cpuset：

```
    # cd /sys/fs/cgroup/cpuset/
    # mkdir cluster
    # mkdir partition
```

> 2. Disable load balancing in the root cpuset to create two new root domains in the CPU sets:

2. 在根cpuset中禁用负载平衡，以在CPU集中创建两个新的根域：

```
    # echo 0 > cpuset.sched_load_balance
```

> 3. Enter the directory for the cluster cpuset, set the CPUs available to 1-7, the memory node the set should run in (in this case the system is not NUMA, so it is always node zero), and set the cpuset to the exclusive mode.

3. 输入群集cpuset的目录，将可用CPU设置为1-7，应在其中运行该内存节点（在这种情况下，系统不是NUMA，因此它始终是零节点），并将cpuset设置为独占模式。

```
    # cd cluster/
    # echo 1-7 > cpuset.cpus
    # echo 0 > cpuset.mems
    # echo 1 > cpuset.cpu_exclusive 
```

> 4. Move all tasks to this CPU set

```
    # ps -eLo lwp | while read thread; do echo $thread > tasks ; done
```

4. 将所有任务移至该CPU集

> 5. Then it is possible to start deadline tasks in this cpuset.

5. 然后可以在此cpuset中启动截止日期任务。

> 6. Configure the partition cpuset:

6. 配置分区cpuset：

```
    # cd ../partition/
    # echo 1 > cpuset.cpu_exclusive 
    # echo 0 > cpuset.mems 
    # echo 0 > cpuset.cpus
```

> 7. Finally move the shell to the partition cpuset.

7. 最后将外壳移动到分区cpuset。

```
    # echo $$ > tasks 
```

> 8. The final step is to run the deadline workload.

8. 最后一步是运行截止期限工作负载。

> With this setup, the task isolated in the partitioned cpuset will not interfere with the tasks in the cluster cpuset, increasing the system’s maximum load while meeting the deadline of real-time tasks.

使用此设置，隔离在分区cpuset中的任务将不会干扰群集cpuset中的任务，从而在满足实时任务的期限的同时增加了系统的最大负载。

## 开发者的观点（The developer’s perspective）

> There are three ways to use the deadline scheduler: as constant bandwidth server, as a periodic/sporadic server waiting for an event, or with a periodic task waiting for replenishment. The most basic parameter for the sched deadline is the period, which defines how often a task is activated. When a task does not have an activation pattern, it is possible to use the deadline scheduler in an aperiodic mode by using only the CBS features.

有三种使用截止期限调度程序的方法：作为恒定带宽服务器，作为等待事件的定期/零星服务器或使用定期任务等待补货。预定期限的最基本参数是期限，它定义了任务激活的频率。当任务没有激活模式时，可以通过仅使用CBS功能以非周期性模式使用截止期限计划程序。

> In the aperiodic case, the best thing the user can do is to estimate how much CPU time a task needs in a given period of time to accomplish the expected result. For instance, if one task needs 200ms each second to accomplish its work, run time would be 200,000,000ns and the period would be 1,000,000,000ns. The [`sched_setattr()`](http://man7.org/linux/man-pages/man2/sched_setattr.2.html) system call is used to set the deadline-scheduling parameters. The following code is a simple example of how to set the mentioned parameters in an application:

在非周期性的情况下，用户可以做的最好的事情是估计任务在给定的时间段内需要多少CPU时间才能达到预期的结果。例如，如果一项任务每秒需要200毫秒才能完成其工作，则运行时间将为200,000,000ns，周期为1,000,000,000ns。所述sched_setattr（） 系统调用用于设置最终期限的调度参数。以下代码是如何在应用程序中设置上述参数的简单示例：

```
    int main (int argc, char **argv)
    {
        int ret;
        int flags = 0;
        struct sched_attr attr;

        memset(&attr, 0, sizeof(attr)); 
        attr.size = sizeof(attr);

        /* This creates a 200ms / 1s reservation */
        attr.sched_policy   = SCHED_DEADLINE;
        attr.sched_runtime  =  200000000;
        attr.sched_deadline = attr.sched_period = 1000000000;

        ret = sched_setattr(0, &attr, flags);
        if (ret < 0) {
            perror("sched_setattr failed to set the priorities");
            exit(-1);
        }

        do_the_computation_without_blocking();
        exit(0);
    }
```

> In the aperiodic case, the task does not need to know when a period starts, and so the task just needs to run, knowing that the scheduler will throttle the task after it has consumed the specified run time.

在非周期性的情况下，任务不需要知道周期的开始时间，因此任务只需要运行即可，因为知道调度程序在消耗了指定的运行时间后将对其进行限制。

> Another use case is to implement a periodic task which starts to run at every periodic run-time replenishment, runs until it finishes its processing, then goes to sleep until the next activation. Using the parameters from the previous example, the following code sample uses the [`sched_yield()`](http://man7.org/linux/man-pages/man2/sched_yield.2.html) system call to notify the scheduler of end of the current activation. The task will be awakened by the next run-time replenishment. Note that the semantics of `sched_yield()` are a bit different for deadline tasks; they will not be scheduled again until the run-time replenishment happens.

另一个用例是实现一个定期任务，该任务在每次定期运行时补充时开始运行，一直运行到完成其处理，然后进入睡眠状态直到下一次激活。使用上一示例中的参数，以下代码示例使用sched_yield（） 系统调用将当前激活结束通知调度程序。下一次运行时补充将唤醒该任务。注意，对于截止日期任务，sched_yield（）的语义有所不同。在运行时补给发生之前，不会再次计划它们。

> Code working in this mode would look like the example above, except that the actual computation looks like:

在这种模式下工作的代码看起来像上面的示例，除了实际的计算看起来像：

```
        for(;;) {
            do_the_computation();
            /* 
	     * Notify the scheduler the end of the computation
             * This syscall will block until the next replenishment
             */
	    sched_yield();
        }
```

> It is worth noting that the computation must finish within the given run time. If the task does not finish, it will be throttled by the CBS algorithm.

值得注意的是，计算必须在给定的运行时间内完成。如果任务未完成，则CBS算法将限制它。

> The most common realtime use case for the realtime task is to wait for an external event to take place. In this case, the task waits in a blocking system call. This system call will wake up the real-time task with, at least, a minimum interval between each activation. That is, it is a sporadic task. Once activated, the task will do the computation and provide the response. Once the task provides the output, the task goes to sleep by blocking waiting for the next event.

实时任务最常见的实时用例是等待外部事件发生。在这种情况下，任务在阻塞的系统调用中等待。该系统调用将以至少两次激活之间的最小间隔唤醒实时任务。也就是说，这是一个零星的任务。激活后，任务将进行计算并提供响应。任务提供输出后，任务将通过阻止等待下一个事件进入睡眠状态。

```
        for(;;) {
            /* 
	     * Block in a blocking system call waiting for a data
             * to be processed.
             */
            process_the_data();
            produce_the_result()
	    block_waiting_for_the_next_event();
        }
```

## 结论（Conclusion）

> The deadline scheduler is able to provide guarantees for realtime tasks based only in the task’s timing constraints. Although global multi-core scheduling faces Dhall’s effect, it is possible to configure the system to achieve a high load utilization using cpusets as a method to partition the systems. Developers can also benefit from the deadline scheduler by designing their application to interact with the scheduler, simplifying the control of the timing behavior of the task.

截止时间调度程序仅根据任务的时间限制就能够为实时任务提供保证。尽管全局多核调度面临Dhall的影响，但可以使用cpusets作为分区系统的方法来配置系统以实现高负载利用率。开发人员还可以通过设计其应用程序与调度程序进行交互，从而简化对任务计时行为的控制，从而从限期调度程序中受益。

> The deadline scheduler tasks have a higher priority than realtime scheduler tasks. That means that even the highest fixed-priority task will be delayed by deadline tasks. Thus, deadline tasks do not need to consider interference from realtime tasks, but realtime tasks must consider interference from deadline tasks.

截止时间调度程序任务比实时调度程序任务具有更高的优先级。这意味着即使是最高固定优先级的任务也将被截止期限任务延迟。因此，截止日期任务不需要考虑来自实时任务的干扰，但是实时任务必须考虑来自截止时间任务的干扰。

> The deadline scheduler and the PREEMPT_RT patch play different roles in improving Linux’s realtime features. While the deadline scheduler allows scheduling tasks in a more predictable way, the PREEMPT_RT patch set improves the kernel by reducing and limiting the amount of time a lower-priority task can delay the execution of a realtime task. It works by reducing the amount of the time a processor runs with preemption and IRQs disabled and the amount of time in which a lower-priority task can delay the execution of a task by holding a lock.

截止日期调度程序和PREEMPT_RT补丁在改善Linux的实时功能方面起着不同的作用。截止期限调度程序允许以更可预测的方式调度任务，而PREEMPT_RT补丁集通过减少和限制优先级较低的任务可以延迟实时任务执行的时间来改善内核。它的工作方式是减少处理器在抢占和禁用IRQ的情况下运行的时间，以及降低优先级较低的任务可以通过保持锁定来延迟任务执行的时间。

> For example, as a realtime task can suffer an activation latency higher than 5ms when running in a non-realtime kernel, it is that this kernel cannot handle deadline tasks with deadlines shorter than 5ms. In contrast, the realtime kernel provides a guarantee, on well tuned and certified hardware, of not delaying the start of the highest priority task by more than 150µs, thus it is possible to handle realtime tasks with deadlines much shorter than 5ms. You can find more about the realtime kernel [here](http://developerblog.redhat.com/?p=425603&preview_id=425603&preview_nonce=28c03def3d&post_format=standard&preview=true).

例如，由于实时任务在非实时内核中运行时可能遭受高于5ms的激活等待时间，因此该内核无法处理期限短于5ms的期限任务。相比之下，实时内核在经过良好调试和认证的硬件上保证不会将最高优先级任务的启动延迟超过150µs，因此可以处理时限远远小于5ms的实时任务。您可以在此处找到有关实时内核的更多信息。

> Acknowledgment: this series of articles was reviewed and improved with comments from Clark Williams, Beth Uptagrafft, Arnaldo Carvalho de Melo, Luis Claudio R. Gonçalves, Oleksandr Natalenko, Jiri Kastner and Tommaso Cucinotta.

致谢：对该系列文章进行了评论和改进，其中包括Clark Williams，Beth Uptagrafft，Arnaldo Carvalho de Melo，Luis Claudio R.Gonçalves，Oleksandr Natalenko，Jiri Kastner和Tommaso Cucinotta的评论。

**了解更多有关 “LWN 中文翻译计划”，请点击 [这里](/lwn/)**

[1]: https://lwn.net/Articles/518993/
[2]: https://en.wikipedia.org/wiki/NP-hardness
