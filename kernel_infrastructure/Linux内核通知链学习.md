# 内核通知链学习

## 一、通知链概述

在linux内核中存在不同的子系统和设备驱动。要想将他们关联起来，比如某个子系统在发生某个事件或其状态发生改变时可以通知使用它服务的其他子系统，这离不开我们接下来要介绍的内核通知链技术。通知链的运作机制包括两个角色：

- **被通知者**：对某一事件感兴趣一方。定义了当事件发生时，相应的处理函数，即回调函数，被通知者将其注册到通知链中（被通知者注册的动作就是在通知链中增加一项）。
- **通知者**：事件的通知者。当检测到某事件，或者本身产生事件时，通知所有对该事件感兴趣的一方事件发生。它定义了一个通知链，其中保存了每一个被通知者对事件的回调函数。通知这个过程实际上就是遍历通知链中的每一项，然后调用相应的回调函数。

包括以下过程：

- 通知者定义通知链。
- 被通知者向通知链中注册回调函数。
- 当事件发生时，通知者发出通知（执行通知链中所有元素的回调函数）。

通过内核通知链，内核中不同的子系统或设备驱动间可以通信。**通知链相关代码位于 `kernel/notifier.c` 中，对应的头文件为 `include/linux/notifier.h` 。下面的代码示例以Linux 6.8为例。**

## 二、数据结构

什么是内核通知链？简单的说它就是一个单链表。在某个子系统中维护这样一个链表，使用它服务的其他模块在这个链表上插上自己的节点，我们把这里的节点叫**通知项**。通知项的结构如下：

```c

typedef	int (*notifier_fn_t)(struct notifier_block *nb,
			unsigned long action, void *data);    // 回调函数类型定义
 
struct notifier_block { 
	notifier_fn_t notifier_call;                  // 回调函数
	struct notifier_block __rcu *next;            // 用于挂到通知链上
	int priority;								  // 优先级
};  
```

其中`notifier_call`是通知链要执行的回调函数，`next`是用来连接其他的通知结构，`priority`是这个通知的优先级，同一条链上的`notifier_block`是按照优先级排列的，数字越大，优先级越高，越会被先执行。

内核围绕核心数据结构`notifier_block`，根据其使用场景的不同，定义了四种内核通知链类型：

**1、原子通知链（ Atomic notifier chains ）**

```c
struct atomic_notifier_head {
        spinlock_t lock;
        struct notifier_block __rcu *head;
};
```

原子通知链**用于中断或原子上下文中，调用者必须在不可睡眠的上下文中调用，因此通知链元素的回调函数不得执行可能阻塞或睡眠的操作**。通知链的表头包含了自旋锁，在其他模块注册/注销该通知链时提供并发保护。

**2、可阻塞通知链（ Blocking notifier chains ）**

```c
struct blocking_notifier_head {
        struct rw_semaphore rwsem;
        struct notifier_block __rcu *head;
};
```

可阻塞通知链**用于进程上下文中，因为其调用路径允许睡眠，回调函数可以执行可能睡眠的操作**。通知链的表头包含了读写信号量，表明使用该通知链通常为读多写少场景，通过读写信号量保证注册/注销和遍历互不干扰。

**3、原始通知链（ Raw notifier chains ）**

```c
struct raw_notifier_head {
        struct notifier_block __rcu *head;
};
```

原始通知链**不对并发访问提供任何内部同步机制。调用者必须在外部自行保证注册、注销和调用过程的同步。通知链的回调函数是否能够睡眠，取决于调用者当前的执行上下文。**表头中不包含任何锁，因为原始通知链的设计目标是**追求最小的性能开销。**

**4、SRCU 通知链（ SRCU notifier chains ）**

```c
struct srcu_notifier_head {
        struct mutex mutex;
        struct srcu_struct srcu;
        struct notifier_block __rcu *head;
};
```

SRCU通知链**用于允许睡眠的上下文中**，其遍历操作基于SRCU（Sleepable RCU）机制保护，从而在**并发注册、注销和调用时无需全局锁即可安全进行。回调函数可以执行可能睡眠的操作**。通知链的表头包含SRCU结构，用于实现高并发、低开销的读侧保护。

## 三、通知链用法

在Linux内核中，无论是哪种类型的通知链，其使用都离不开以下基本操作：**初始化链表头，注册通知项（向链表添加元素），注销通知项（从链表删除元素）以及调用通知链（遍历链表并执行回调）**。

其中，后三个操作接口都是基于通知链最基本的注册、注销和遍历接口实现，并结合具体的通知链类型进行封装后得到的。因此，掌握一种通知链的接口实现，即可类推理解其他类型的通知链。

接下来我们就以常用的原子通知链为例进行讲解：

### 1、初始化通知链

初始化通知链有两种方式：

##### 1.1、静态定义并初始化

使用的宏为`ATOMIC_NOTIFIER_HEAD`，表示**定义了名称为name的原子通知链对象并对其完成了初始化**。通常适用于在全局或静态上下文中。

##### 1.2、动态初始化

使用的宏为`ATOMIC_INIT_NOTIFIER_HEAD`，用于**对已存在的 atomic_notifier_head 变量name进行初始化**，通常适用于在 **动态创建结构体实例** 或 **结构体成员中包含通知链表头** 的情况，作为开发用的更多。

##### 1.3、代码实例

```c
// 静态定义并初始化
#define ATOMIC_NOTIFIER_INIT(name) {				\
		.lock = __SPIN_LOCK_UNLOCKED(name.lock),	\
		.head = NULL }

#define ATOMIC_NOTIFIER_HEAD(name)				\
	struct atomic_notifier_head name =			\
		ATOMIC_NOTIFIER_INIT(name)

// 动态初始化
#define ATOMIC_INIT_NOTIFIER_HEAD(name) do {	\
		spin_lock_init(&(name)->lock);	\
		(name)->head = NULL;		\
	} while (0)
```

这两种宏最终都等价于：

```	c
#define ATOMIC_NOTIFIER_INIT(name) {				\
		.lock = __SPIN_LOCK_UNLOCKED(name.lock),	\
		.head = NULL }
```

### 2、注册/注销通知项

初始化完成后，其他模块才能将自己的“监听”函数注册进通知链。当模块退出或不再需要接收事件时，需要将对应项注销。	

##### 2.1、注册通知项

```c
// 原子通知链对外注册接口
int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
		struct notifier_block *n)
{
	unsigned long flags;
	int ret;

	spin_lock_irqsave(&nh->lock, flags);
	ret = notifier_chain_register(&nh->head, n, false);
	spin_unlock_irqrestore(&nh->lock, flags);
	return ret;
}

// 底层逻辑
static int notifier_chain_register(struct notifier_block **nl,
				   struct notifier_block *n,
				   bool unique_priority)
{
	while ((*nl) != NULL) {
		if (unlikely((*nl) == n)) { // 若该通知项已经注册则返回，避免重复注册
			WARN(1, "notifier callback %ps already registered",
			     n->notifier_call);
			return -EEXIST;
		}
		if (n->priority > (*nl)->priority)
			break;
		if (n->priority == (*nl)->priority && unique_priority) // unique_priority表示拒绝重复优先级
			return -EBUSY;
		nl = &((*nl)->next);
	}
	n->next = *nl;
	rcu_assign_pointer(*nl, n);
	trace_notifier_register((void *)n->notifier_call);
	return 0;
}
```

如上所示，主要机制如下：

- 外层使用自旋锁防止多个写者并发修改
- 内层通过`rcu_assign_pointer`提供 release 语义的发布操作，确保读者在通过 `rcu_dereference_raw()` 访问时能看到链表中节点的完整已初始化状态
- 插入顺序遵循优先级从大到小，无优先级则遵循先来后到的顺序

##### 2.2、注销通知项

```c
// 原子通知链对外注销接口
int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,
		struct notifier_block *n)
{
	unsigned long flags;
	int ret;

	spin_lock_irqsave(&nh->lock, flags);
	ret = notifier_chain_unregister(&nh->head, n);
	spin_unlock_irqrestore(&nh->lock, flags);
	synchronize_rcu();
	return ret;
}

// 底层逻辑
static int notifier_chain_unregister(struct notifier_block **nl,
		struct notifier_block *n)
{
	while ((*nl) != NULL) {
		if ((*nl) == n) {
			rcu_assign_pointer(*nl, n->next);
			trace_notifier_unregister((void *)n->notifier_call);
			return 0;
		}
		nl = &((*nl)->next);
	}
	return -ENOENT;
}
```

如上所示，主要机制如下：

- 使用自旋锁实现写保护
- 注销后调用 `synchronize_rcu()`，等待所有正在执行的 RCU 读者退出。**这里可能会导致阻塞，该注销接口不宜在中断上下文频繁调用**

### 3、调用通知链

当事件发生时，模块需通知所有注册的监听者，这时会调用：

```c
// 原子通知链对外调用接口
int atomic_notifier_call_chain(struct atomic_notifier_head *nh,
			       unsigned long val, void *v)
{
	int ret;

	rcu_read_lock();
	ret = notifier_call_chain(&nh->head, val, v, -1, NULL);
	rcu_read_unlock();

	return ret;
}

// 底层逻辑
static int notifier_call_chain(struct notifier_block **nl,
			       unsigned long val, void *v,
			       int nr_to_call, int *nr_calls)
{
	int ret = NOTIFY_DONE;
	struct notifier_block *nb, *next_nb;

	nb = rcu_dereference_raw(*nl); // 读取第一个节点

	while (nb && nr_to_call) {
		next_nb = rcu_dereference_raw(nb->next);

		trace_notifier_run((void *)nb->notifier_call);
		ret = nb->notifier_call(nb, val, v); // 调用通知项的回调函数

		if (nr_calls)
			(*nr_calls)++;

		if (ret & NOTIFY_STOP_MASK) // 回调返回NOTIFY_STOP_MASK,提前终止遍历
			break;
		nb = next_nb;
		nr_to_call--;
	}
	return ret;
}
```

如上所示，主要机制如下：

- 遍历置于RCU读临界区，**所以回调函数中不得阻塞或睡眠**
- 通过 `rcu_dereference_raw()` 获取节点（acquire 语义，匹配注册侧的 release）
- `val`通常为发生的事件枚举，`v`通常是上下文数据

其他三类通知链和原子通知链的原理相似，主要差异点就是对注册、注销和调用通知链的三个基本接口的封装以及各自的使用场景。

### 4、实际驱动用例

在上面我们以原子通知链为例，梳理了申明、注册、注销和调用接口的实现机制。实际上内核各子系统在使用原子通知链时，还会对我们已知的四种类型通知链再进行一次包装，以形成各自的特色。

在网络子系统中，除了最常见的**netdev_chain（网络设备状态变化通知链）**，还有针对**IPv4 地址变化的inetaddr_chain**以及**IPv6 地址变化的 inet6addr_chain**。当通过`ifconfig`或`ip addr add/del`对某设备的IP地址进行增删时，就会触发`inetaddr_chain`。

其注册与调用机制如下：

```c
// ipv4地址变化通知链注册接口
int register_inetaddr_notifier(struct notifier_block *nb)
{
	return blocking_notifier_chain_register(&inetaddr_chain, nb);
}
EXPORT_SYMBOL(register_inetaddr_notifier);

// ipv4地址变化通知链注销接口
int unregister_inetaddr_notifier(struct notifier_block *nb)
{
	return blocking_notifier_chain_unregister(&inetaddr_chain, nb);
}
EXPORT_SYMBOL(unregister_inetaddr_notifier);

// 当某个IPv4地址被删除时调用接口
blocking_notifier_call_chain(&inetaddr_chain, NETDEV_DOWN, ifa);
```

许多子系统（如 RDMA/InfiniBand 栈）都会注册该通知链，以在IP地址变化时同步更新自身状态。

例如RDMA子系统在初始化时就针对该通知链注册了自己的回调`inetaddr_event`。当接收到`NETDEV_UP`或`NETDEV_DOWN`（指地址增加/删除）事件时，**根据事件类型更新 RDMA 设备的 GID 表，使网络层与 RDMA 层的地址映射保持一致**。

## 四、注意事项

1. 通知链调用是同步执行的，任何一个回调函数执行速度直接影响通知链传播延迟，因此通常对耗时较长的操作，最好是在工作队列中异步执行。
2. 关于通知链的回调函数，正常情况下都需要返回`NOTIFY_OK`或者NOTIFY_DONE` ，这样通知链上后面挂载的其他函数可以继续执行。如果返回 `NOTIFY_STOP` ，则会使得通知链上后续挂载的函数无法得到执行。
3. 走读内核代码时，不知道某个模块的通知链作用是什么，可通过其调用通知链时传的事件间接推断
