## About spinlock
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
	arch_spinlock_t raw_lock; //根据不同的arch有定义为volatile unsigned int/int/u32
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
```
### spinlock基本用法
spinlock api头文件
include/linux/spinlock.h
使用基本步骤是：
1.初始化自旋锁  
    定义一个自旋锁 spinlock_t mylock = SPIN_LOCK_UNLOCKED  
    或者调用spin_lock_init函数对锁进行初始化
2.进入临界区之前调用api获取锁
    spin_lock(spinlock_t *lock);
  这个过程不可中断，一旦调用了spin_lock在获取锁之前将一直处于自旋状态
3.释放自旋锁
     spin_unlock(spinlock_t *lock);
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
#### spin_unlock函数
```c

```
_____________________________________
### 背景知识
UP（Uni-Processor）：系统只有一个处理器单元，即单核CPU系统。

SMP（Symmetric Multi-Processors）：系统有多个处理器单元。各个处理器之间共享总线，内存等等。在操作系统看来，各个处理器之间没有区别。

要注意，这里提到的“处理器单元”是指“logic CPU”，而不是“physical CPU”。举个例子，如果一个“physical CPU”包含2个core，并且一个core包含2个hardware thread。则一个“处理器单元”就是一个hardware thread。
### 参考
[1]Linux Device Drivers
[2]https://zhuanlan.zhihu.com/p/96001570 Linux中的Barrier
[3]https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652664399&idx=1&sn=2dc274f63f2136dec7166df24d94d300&scene=21#wechat_redirect
[4]https://blog.csdn.net/joker0910/article/details/7782765 这篇博客写的很好
[5]http://www.wowotech.net/kernel_synchronization/spinlock.html 这篇博客非常系统
[6]https://www.jianshu.com/p/f0d6e7103d9b

