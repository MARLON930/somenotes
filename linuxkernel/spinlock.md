## About spinlock
### 背景知识
UP（Uni-Processor）：系统只有一个处理器单元，即单核CPU系统。
SMP（Symmetric Multi-Processors）：系统有多个处理器单元。各个处理器之间共享总线，内存等等。在操作系统看来，各个处理器之间没有区别。
要注意，这里提到的“处理器单元”是指“logic CPU”，而不是“physical CPU”。举个例子，如果一个“physical CPU”包含2个core，并且一个core包含2个hardware thread。则一个“处理器单元”就是一个hardware thread。<br>
spinlock发展至今经历了以下几个阶段：
 第一个版本 其实，kernel很早就实现了spinlock，之前的lock叫做ticket-spinlock，最早的ticket lock非常简单，自旋锁spinlock用一个整数值来表示，表明了锁是否可用，初值设为1。spin_lock()函数通过递减val（原子方式），然后查看是否为0，若为0则成功拿锁，若为负数则代表锁已属于他人，所以它进入spin状态，不断查询val值直到变为1。当锁的拥有者完成critical section的执行，将val置为1，即释放锁。这是最原始的实现，实际上就是原子变量，坏处在于没有排队机制，等待时间最长的CPU不一定能最先拿到锁。<br>
第二个版本 后来，就是最常见的owner与next叫号机制实现的spinlock，这种机制的问题在于假设一个CPU持锁，7个CPU等待，等待的7个CPU都必须不停地读cache，当持锁者解锁，只有最先等待的CPU能够拿到，与其他的CPU无关，但是所有等待的CPU都必须重新刷新cache，因为解锁的时候会修改结构体成员，数据发生了变化。<br>
第三个版本 考虑到这种性能浪费，就有人设计另一种锁 MCS-lock：为等待的CPU分配锁的副本，每个CPU只等待自己所属的副本发生改变（下文会把这种情况叫做在自己的副本自旋），这样避免cache刷新的问题。这种锁始终没有在kernel得到广泛使用（只有x86会有它的接口），因为他的结构里面多了一个指针导致结构体变大了，多了一个指针成员，而spinlock很多情况下都是内嵌在一些结构体里面使用，这样必然影响很多结构体的大小，内核对某些结构体的大小非常敏感，所以此方案几乎无人使用。<br>
 ARM 4.16内核之后使用的版本 最终，终于有人将MCS的结构压缩，在没有影响结构体大小的情况下实现【副本自旋】这一思想，这就是qspin-lock.<br>
### spinlock_t数据结构定义 
include/linux/spinlock_types.h
```c
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t;

```
#### raw_spinlock_t数据结构
```c
typedef struct raw_spinlock {
	arch_spinlock_t raw_lock; //根据不同的arch有定义为volatile unsigned int/int/u32/qspinlock
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;


typedef struct qspinlock {
	union {
		atomic_t val;

		/*
		 * By using the whole 2nd least significant byte for the
		 * pending bit, we can allow better optimization of the lock
		 * acquisition for the pending bit holder.
		 */
#ifdef __LITTLE_ENDIAN
		struct {
			u8	locked;
			u8	pending;
		};
		struct {
			u16	locked_pending;
			u16	tail;
		};
#else
		struct {
			u16	tail;
			u16	locked_pending;
		};
		struct {
			u8	reserved[2];
			u8	pending;
			u8	locked;
		};
#endif
	};
} arch_spinlock_t;
```
### spinlock基本用法
spinlock api头文件
include/linux/spinlock.h
使用基本步骤是：
<br>1.初始化自旋锁
<br>    定义一个自旋锁 spinlock_t mylock = SPIN_LOCK_UNLOCKED
<br>    或者调用spin_lock_init函数对锁进行初始化
<br>2.进入临界区之前调用api获取锁
<br>    spin_lock(spinlock_t *lock);
<br>  这个过程不可中断，一旦调用了spin_lock在获取锁之前将一直处于自旋状态
<br>3.释放自旋锁
<br>    spin_unlock(spinlock_t *lock);
#### spin_lock_init函数
spin_lock_init(spinlock_t *lock);
```c
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}

#define spin_lock_init(_lock)				\
do {							\
	spinlock_check(_lock);				\
	raw_spin_lock_init(&(_lock)->rlock);		\
} while (0)

#ifdef CONFIG_DEBUG_SPINLOCK
  extern void __raw_spin_lock_init(raw_spinlock_t *lock, const char *name,
				   struct lock_class_key *key);
# define raw_spin_lock_init(lock)				\
do {								\
	static struct lock_class_key __key;			\
								\
	__raw_spin_lock_init((lock), #lock, &__key);		\
} while (0)

#else
# define raw_spin_lock_init(lock)				\
	do { *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock); } while (0)
#endif

// kernel/locking/spinlock_debug.c
void __raw_spin_lock_init(raw_spinlock_t *lock, const char *name,
			  struct lock_class_key *key)
{
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/*
	 * Make sure we are not reinitializing a held lock:
	 */
	debug_check_no_locks_freed((void *)lock, sizeof(*lock));
	lockdep_init_map(&lock->dep_map, name, key, 0);
#endif
	lock->raw_lock = (arch_spinlock_t)__ARCH_SPIN_LOCK_UNLOCKED;
	lock->magic = SPINLOCK_MAGIC;
	lock->owner = SPINLOCK_OWNER_INIT;
	lock->owner_cpu = -1;
}
#define __RAW_SPIN_LOCK_INITIALIZER(lockname)	\
	{					\
	.raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,	\
	SPIN_DEBUG_INIT(lockname)		\
	SPIN_DEP_MAP_INIT(lockname) }

#define __RAW_SPIN_LOCK_UNLOCKED(lockname)	\
	(raw_spinlock_t) __RAW_SPIN_LOCK_INITIALIZER(lockname)
```
#### spin_lock函数
```c
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
#define raw_spin_lock(lock)	_raw_spin_lock(lock)

#ifndef CONFIG_INLINE_SPIN_LOCK
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}
EXPORT_SYMBOL(_raw_spin_lock);
#endif

static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}

```
可以看到实际上spin_lock先关调用preempt_disable()闭了内核抢占。
```c
#define preempt_disable() \
do { \
	preempt_count_inc(); \
	barrier(); \
} while (0)

```
spin_lock()会调用preempt_disable() 导致本核的抢占调度被关闭(preempt_disable函数实际增加preempt_count来达到此效果)。
这里用到了barrier内存屏障
```c
#define barrier() __asm__ __volatile__("": : :"memory")
```
这里的"memory"就是告知gcc，在汇编代码中，我修改了内存中的内容，之前的C代码块和之后的C代码块看到的内存是不一样的，对内存的访问不能依赖于嵌入汇编之前的C代码块中寄存器的内容，需要重新从内存中读取变量的值来刷新寄存器。
关闭内核抢占之后
```c
#define lock_acquire_exclusive(l, s, t, n, i)		lock_acquire(l, s, t, 0, 1, n, i)
#define spin_acquire(l, s, t, i)		lock_acquire_exclusive(l, s, t, NULL, i)
```
UP中很简单，本质上就是一个preempt_disable而已，和我们在第二章中分析的一致。SMP中稍显复杂，preempt_disable当然也是必须的，spin_acquire可以略过，这是和运行时检查锁的有效性有关的，如果没有定义CONFIG_LOCKDEP其实就是空函数。如果没有定义CONFIG_LOCK_STAT（和锁的统计信息相关），LOCK_CONTENDED就是调用do_raw_spin_lock而已，如果没有定义CONFIG_DEBUG_SPINLOCK，它的代码如下
```c
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
    __acquire(lock);
    arch_spin_lock(&lock->raw_lock);
}
```
在arm架构中实现
```c
/*
 * ARMv6 ticket-based spin-locking.
 *
 * A memory barrier is required after we get a lock, and before we
 * release it, because V6 CPUs are assumed to have weakly ordered
 * memory.
 */

static inline void arch_spin_lock(arch_spinlock_t *lock)
{
	unsigned long tmp;
	u32 newval;
	arch_spinlock_t lockval;

	prefetchw(&lock->slock);		//---1
	__asm__ __volatile__(
"1:	ldrex	%0, [%3]\n"    			//---2
"	add	%1, %0, %4\n"				
"	strex	%2, %1, [%3]\n"			//---3
"	teq	%2, #0\n"					//---4
"	bne	1b"						
	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp)	
	: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
	: "cc");										

	while (lockval.tickets.next != lockval.tickets.owner) { //---5
		wfe();												//---6
		lockval.tickets.owner = READ_ONCE(lock->tickets.owner); //---7
	}

	smp_mb(); //---8
}
```
#### spin_unlock函数
```c

#define raw_spin_unlock(lock)		_raw_spin_unlock(lock)
void __lockfunc _raw_spin_unlock(raw_spinlock_t *lock)
{
	__raw_spin_unlock(lock);
}
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
	spin_release(&lock->dep_map, _RET_IP_); // ----1
	do_raw_spin_unlock(lock);				// ----2
	preempt_enable();						// ----3
}
```

_____________________________________


### 参考
[1]Linux Device Drivers<br>
[2]https://zhuanlan.zhihu.com/p/96001570 Linux中的Barrier<br>
[3]https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652664399&idx=1&sn=2dc274f63f2136dec7166df24d94d300&scene=21#wechat_redirect<br>
[4]https://blog.csdn.net/joker0910/article/details/7782765 这篇博客写的很好<br>
[5]http://www.wowotech.net/kernel_synchronization/spinlock.html 这篇博客非常系统<br>
[6]https://www.jianshu.com/p/f0d6e7103d9b<br>
[7]https://blog.csdn.net/feisezaiyue/article/details/106171499<br>
[8]http://libfbp.blogspot.com/2018/01/c-mellor-crummey-scott-mcs-lock.html
