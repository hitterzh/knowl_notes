## linux kernel

### Task管理：

task_struct构成：

/* CPU-specific state of this task */
	struct thread_struct thread;



/* filesystem information */
	struct fs_struct *fs;*

/* open file information */
	struct files_struct *files;



/* signal handlers */
	struct signal_struct *signal;
	struct sighand_struct *sighand;

	sigset_t blocked, real_blocked;
	sigset_t saved_sigmask;	/* restored if set_restore_sigmask() was used */
	struct sigpending pending;




#### 1. Task管理结构

#### 2. Task的生命周期

#### 3. Task调度

#### 4. 上下文切换



### 第三章 进程

#### 1. 进程、轻量级进程、线程

多数多线程application使用ptread标准库接口实现

linuxTreads、NPTL、NGPT？

Linux提供lightweight process对多线程application提供支持，共享资源：地址空间、fds （共享资源意味着需要同步）



#### 2. 进程描述符

task_struct位于sched.h



六大部分构成：**TODO**

- ​	**thread_info: 进程基本信息？**
- ​	**mm_struct**
- ​	**tty_struct:**
- ​	**fs_struct:?**
- ​	**files_struct:**
- ​	**signal_struct:**



#### 3. 进程状态

```c
/*
 * Task state bitmask. NOTE! These bits are also
 * encoded in fs/proc/array.c: get_task_state().
 *
 * We have two separate sets of flags: task->state
 * is about runnability, while task->exit_state are
 * about the task exiting. Confusing, but this way
 * modifying one set can't modify the other one by
 * mistake.
 */
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2
#define __TASK_STOPPED		4
#define __TASK_TRACED		8
/* in tsk->exit_state */
#define EXIT_ZOMBIE		16
#define EXIT_DEAD		32
/* in tsk->state again */
#define TASK_DEAD		64
#define TASK_WAKEKILL		128
#define TASK_WAKING		256
#define TASK_PARKED		512
#define TASK_STATE_MAX		1024
```

**TODO：状态分两个flag，退出状态这么多？**



#### 4. 进程标识

pid默认32767大小，可以修改(最大4million)。（但是进程数量多，调度的开销会很大，无意义）

**pid的分配管理**（pidmap_init初始化）: 

```
/*
 * PID-map pages start out as NULL, they get allocated upon
 * first use and are never deallocated. This way a low pid_max
 * value does not cause lots of bitmaps to be allocated, but
 * the scheme scales to up to 4 million PIDs, runtime.
 */
struct pid_namespace init_pid_ns = {
	.kref = {
		.refcount       = ATOMIC_INIT(2),
	},
	.pidmap = {
		[ 0 ... PIDMAP_ENTRIES-1] = { ATOMIC_INIT(BITS_PER_PAGE), NULL }
	},
	.last_pid = 0,
	.level = 0,
	.child_reaper = &init_task,
	.user_ns = &init_user_ns,
	.proc_inum = PROC_PID_INIT_INO,
};
```

posix规定一组线程拥有相同PID，**signal可以发往一组线程？ 所以getpid()获取的是tgid**

thread_info存放于task内核栈底，有成员指向task_struct，**current指针的实现。**（多核的话，current定义成数组？没必要，返回当前cpu上运行task）（早期未将thread_info放于栈底时，current需要数组）



#### 5. 进程管理

进程链表：

链表头为**init_task**上的list，0号process/swapper?的结构，全局变量

运行链表（TODO：涉及sched，相关目录、源码、接口）：

```
/*
 * This is the priority-queue data structure of the RT scheduling class:
 */
struct rt_prio_array {
	DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1); /* include 1 bit for delimiter */
	struct list_head queue[MAX_RT_PRIO];
};
```

```
/*
 * This is the main, per-CPU runqueue data structure.
 *
 * Locking rule: those places that want to lock multiple runqueues
 * (such as the load balancing or the thread migration code), lock
 * acquire operations must be ordered by ascending &runqueue.
 */
struct rq

static void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
	update_rq_clock(rq);
	sched_info_queued(p);
	p->sched_class->enqueue_task(rq, p, flags);
}
```



**进程关系：**

1. 进程亲属关系：

```
	/*
	 * pointers to (original) parent process, youngest child, younger sibling,
	 * older sibling, respectively.  (p->father can be replaced with
	 * p->real_parent->pid)
	 */
	struct task_struct __rcu *real_parent; /* real parent process */
	struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */什么时候与real不一样呢？ptrace时
	/*
	 * children/sibling forms the list of my natural children
	 */
	struct list_head children;	/* list of my children */
	struct list_head sibling;	/* linkage in my parent's children list */
```

2. 非亲属关系：？TODO

   ```
   struct task_struct *group_leader;	/* threadgroup leader */
   	 * ptraced is the list of tasks this task is using ptrace on.
   	 * This includes both natural children and PTRACE_ATTACH targets.
   	 * p->ptrace_entry is p's link on the p->parent->ptraced list.
   	 */
   	struct list_head ptraced;
   	struct list_head ptrace_entry;
   enum pid_type
   {
   	PIDTYPE_PID,
   	PIDTYPE_PGID,
   	PIDTYPE_SID,
   	PIDTYPE_MAX
   };
   ```



PIDhash表和链表

查找：4个hash表

pid.c TODO？



