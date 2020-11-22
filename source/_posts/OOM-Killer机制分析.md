---
title: OOM Killer机制分析
date: 2020-11-21 23:36:25
categories: 技术
tags: linux
---

近日上班工作群有人说，隔壁游戏组突然一连挂了好几台服务器，很突然暂时没分析出宕机的原因。确实很诡异，因为服务器接连挂掉的情况并不常见。我们组一下子也紧张起来，因为我们两个组用的是同一个游戏服务器引擎，隔壁出事了，也预示着我们也有类似的风险。不过没多久，隔壁组宕机的原因定位到了，有位开发在执行扫档脚本时，因为脚本grep的筛选条件没写好，这个脚本进程吃掉了系统几十G内存，触发了Linux操作系统的**OOM Killer机制**，把游戏进程给杀死了。因为这位开发执行的是全服扫档（几百个服务器同时执行脚本），因此接连有服务器宕机。好在发现的早，不然某知名游戏全服宕机的新闻将刷爆游戏论坛。

情景还原：我需要去游戏服去扫档（也就是扫log）获得一些游戏数据，我写了一个脚本打算把符合条件的数据重定向到一个新文件，但是太大意，脚本没写好，条件没写好，写成了类似的：`grep * log/*`，然而我们log里的文件有几百G，这样会导致会把所有log读入内存，因此迅速吃掉系统的大量内存，然后游戏进程就挂了。

听起来还是觉得很不可思议：我去扫log居然把线上业务搞挂了，真是倒霉透了。其实有更倒霉的，我们组有组员曾经因为用vim打开一个超大的文件，也把内存大量吃掉，同样导致了游戏宕机。后来运维做了一些措施限制vim打开一些超大文件（文件太大就不让打开了），防止这类似情况的发生。显然隔壁组没在这块吃过亏，因此造成本次事故。

这是一个真实生产下Linux OOM Killer机制对线上业务造成恶劣影响的典型例子，因此这里想深入讨论一下OOM Killer机制。这个线上事故，即使听到了最终原因分析，但心中仍然有很多疑问：**短时间内快速吃掉大量内存的是脚本进程，为什么系统不去杀死脚本进程，而是杀死占用内存一直稳定的游戏进程？**



## 什么是OOM Killer

OOM全称 Out-of-Memory，也就是操作系统的可利用的内存已经不足了，没法再分配新的内存出来给进程，导致系统没法继续工作，如果不紧急处理，最终的结果必定是系统关机，系统上的所有进程将被杀死。因此OS为了保证内核系统层面的稳定运行，就会根据一定算法规则，选出最应该优先被杀死的进程（理论上就是最占内存空间的那个进程）进行杀死，杀死之后系统就腾出了大量的内存空间，系统的生命将得以延续，继续稳定运行，而这个机制就是OOM Killer机制。



读到这里其实会对Linux的这个设计产生比较多的疑问，最大的疑问就是：为什么要杀死无辜者进程，而不处理肇事者进程？

仔细再分析应该就明白了，**其实操作系统没办法去区分哪个是肇事者进程**。考虑这种情况：A进程疯狂分配内存，导致系统可用的内存已用完，此时B进程需要分配内存空间，那B进程肯定分配空间失败啊，不断抛出异常，挂掉的还是无辜的B;如果是OS自己需要分配内存空间，发现alloc_page返回失败，那就严重多了，为了不影响后面的逻辑，OS会更倾向于与fast fail，选择shutdown。



翻了下linux OOM Killer的[源码](https://github.com/torvalds/linux/blob/master/mm/oom_kill.c)，看到作者留下的这句话，还是蛮有意思的。

```
 * out_of_memory - kill the "best" process when we run out of memory
 * If we run out of memory, we have the choice between either
 * killing a random task (bad), letting the system crash (worse)
 * OR try to be smart about which process to kill. Note that we
 * don't have to be perfect here, we just have to be good.
```

大概翻译一下：“当我们内存不足时，我们有两种处理方案：随机杀死一个任务，这可能会导致系统崩溃；或者尝试有策略地选出值得杀死的那个任务。我们没有必要做到最好，但我们只需尽力把这事做好”。-- 潜台词：这件事我们至今都没找到一个完美算法，误杀进程我们不背锅。

继续从源码分析oom killer。进程使用__alloc_pages()分配内存时，当分配失败时，如果系统有配置OOM Killer，就会进入这个out_of_memory的逻辑。这个out_of_memory做了这几件事：选择要杀死的进程、杀死进程、回收进程内存空间。

```c
bool out_of_memory(struct oom_control *oc)
{
	unsigned long freed = 0;

	if (oom_killer_disabled)
		return false;

	if (!is_memcg_oom(oc)) {
		blocking_notifier_call_chain(&oom_notify_list, 0, &freed);
		if (freed > 0)
			/* Got some memory back in the last second. */
			return true;
	}

	/*
	 * If current has a pending SIGKILL or is exiting, then automatically
	 * select it.  The goal is to allow it to allocate so that it may
	 * quickly exit and free its memory.
	 */
	 // task_will_free_mem函数其实是去检查一下当前有没有进程挂了，有的话就回收他的内存，回收内存后那就不需要oom killer杀进程了
	if (task_will_free_mem(current)) {
		mark_oom_victim(current);
		wake_oom_reaper(current);
		return true;
	}

	/*
	 * The OOM killer does not compensate for IO-less reclaim.
	 * pagefault_out_of_memory lost its gfp context so we have to
	 * make sure exclude 0 mask - all other users should have at least
	 * ___GFP_DIRECT_RECLAIM to get here. But mem_cgroup_oom() has to
	 * invoke the OOM killer even if it is a GFP_NOFS allocation.
	 */
	if (oc->gfp_mask && !(oc->gfp_mask & __GFP_FS) && !is_memcg_oom(oc))
		return true;

	/*
	 * Check if there were limitations on the allocation (only relevant for
	 * NUMA and memcg) that may require different handling.
	 */
	oc->constraint = constrained_alloc(oc);
	if (oc->constraint != CONSTRAINT_MEMORY_POLICY)
		oc->nodemask = NULL;
	check_panic_on_oom(oc);

	if (!is_memcg_oom(oc) && sysctl_oom_kill_allocating_task &&
	    current->mm && !oom_unkillable_task(current) &&
	    oom_cpuset_eligible(current, oc) &&
	    current->signal->oom_score_adj != OOM_SCORE_ADJ_MIN) {
		get_task_struct(current);
		oc->chosen = current;
		oom_kill_process(oc, "Out of memory (oom_kill_allocating_task)");
		return true;
	}

	select_bad_process(oc);  //选择需要杀死的进程
	/* Found nothing?!?! */  // 找不到可以杀死的进程，那操作系统死锁了，这个情况下系统必然挂了
	if (!oc->chosen) {
		dump_header(oc, NULL);
		pr_warn("Out of memory and no killable processes...\n");
		/*
		 * If we got here due to an actual allocation at the
		 * system level, we cannot survive this and will enter
		 * an endless loop in the allocator. Bail out now.
		 */
		if (!is_sysrq_oom(oc) && !is_memcg_oom(oc))
			panic("System is deadlocked on memory\n");
	}
	if (oc->chosen && oc->chosen != (void *)-1UL)
		oom_kill_process(oc, !is_memcg_oom(oc) ? "Out of memory" :
				 "Memory cgroup out of memory");
	return !!oc->chosen;
}


```



oom killer在选择进程杀死前先去检查一下当前有没有进程挂了，有的话优先回收他的内存资源，毕竟杀死一个线上运行的进程毕竟是下下策。然后才是进入到进程选择阶段，oom_kill.c这样实现了选择策略：

```c
long oom_badness(struct task_struct *p, unsigned long totalpages)
{
	long points;
	long adj;

	if (oom_unkillable_task(p))
		return LONG_MIN;

	p = find_lock_task_mm(p);
	if (!p)
		return LONG_MIN;

	/*
	 * Do not even consider tasks which are explicitly marked oom
	 * unkillable or have been already oom reaped or the are in
	 * the middle of vfork
	 */
	adj = (long)p->signal->oom_score_adj;
	if (adj == OOM_SCORE_ADJ_MIN ||
			test_bit(MMF_OOM_SKIP, &p->mm->flags) ||
			in_vfork(p)) {
		task_unlock(p);
		return LONG_MIN;
	}

	/*
	 * The baseline for the badness score is the proportion of RAM that each
	 * task's rss, pagetable and swap space use.
	 */
	points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) +
		mm_pgtables_bytes(p->mm) / PAGE_SIZE;
	task_unlock(p);

	/* Normalize to oom_score_adj units */
	adj *= totalpages / 1000;
	points += adj;

	return points;
}

```



Oom killer通过这个oom_badness函数进行打分，返回值是根据一定策略给进程打的分数，后续oom killer根据该分数高低选择出最该杀死的那个进程（分数越高越优先杀死），这里需要注意3个点：

1. `adj == OOM_SCORE_ADJ_MIN`时，说明该进程已被设置为不可被杀死进程，返回的得分将无限低（LONG_MIN）。
2. `points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) + mm_pgtables_bytes(p->mm) / PAGE_SIZE;`分数公式，分数是由这三部分计算打出：进程所占用的内存中的空间、SWAP所占用的空间、page cache里所占用的空间 ；
3. `adj *= totalpages / 1000;  points += adj;` 分数另一部分的构成是这个oom_score_adj，这个是配置在内核文件的，范围是-1000~1000，默认是0，所以oom_badness先把该分数归一化，再做加法。

所以，Linux提供了一个策略，可以让用户通过填写oom_score_adj文件来影响oom killer的选择，**当你填-1000时，则表示该进程将不会被杀死**，但如果你填写的是非-1000，那这个进程还是会参与打分，但会受到oom_score_adj的影响，比如oom_score_adj你填了-3，当你的进程消耗内存很大时，同样大概率会被杀死。还有一个值得注意的是，**oom killer选择策略，只受进程占用内存和oom_score_adj的影响，至于该进程是否是短时间内快速吃掉大量内存，oom killer并不关心**。这就很好解释了，为什么我们游戏进程（占用内存最多但稳定）会被优先杀死了。oom_score_adj的值可以在/proc/<pid>/oom_score_adj上修改。



读到这里也大概明白oom killer的设计思路了，系统也希望有一个策略能完美地选出该杀的进程，但现实上却没有这么优秀的算法，跑在OS上的进程成千上万，我们怎么能这么有把握选出的进程必然是正确的，只是选出一个比较合适的而已，误杀在所难免。举个现实的例子，你在你的macbook上开着QQ音乐听歌，同时也开着matlab做实验，QQ音乐占用内存不多，matlab占用内存大，此时我们在matlab运行某个算法导致吃掉大量内存触发OOM Killer，那问题来了，此时你是OS，你会选择杀死matlab还是QQ音乐？从OS的角度，MATLAB是短时间内快速吃掉大量内存而且是当前内存占用最多的进程，理应杀他；用户角度，MATLAB还跑着实验啊，实验数据还没保存，你杀QQ音乐啊，那个进程对我没什么用。所以，一个优秀的策略还真不能覆盖全部场景。



生产环境上也如此，脚本进程和游戏进程，同样是占用大量内存的进程，在没额外的判断条件下，杀死任何之一都是符合要求，更何况此时游戏进程占用了更多的内存，即使是短时间内快速消耗内存的脚本进程，在OS看来，也许真没必要杀死，在OS的角度，杀死后快速腾出大量空间进程，才是那个最值得杀的进程。



## 怎么避免OOM Killer误杀我的业务进程？

###### 避免oom killer的方案

- 直接修改/proc/<pid>/oom_score_adj文件，将其置为-1000

  ```
  以前是通过/proc/<pid>/oom_score来控制的，但近年来新版linux已经使用oom_score_adj来代替旧版的oom_score，
  
  参考：https://github.com/tinganho/linux-kernel/blob/master/Documentation/feature-removal-schedule.txt#L171
  ```

  

- 直接关闭oom-killer 

  ```
  # echo "0" > /proc/sys/vm/oom-kill 关闭
  # echo "1″ > /proc/sys/vm/oom-kill  激活
  ```

  





