## 关于condition variable的那些事儿

申明：本文仅讨论cpp的相关实现，不涉及Java等其他语言。

之前对于condition variable的使用仅限于“照方抓药”的阶段，一直有一些疑问但是未深究过，这篇文章主要基于condition variable有个初步认知能够抓药的基础上，深究一些我心中的疑问点。

本文的行文主要关注的问题：

- wait&notify的正常逻辑
- 同一个con_var wait不同的mutex，会发生什么呢?
- wait之前lock mutex，那么notify需要lock mutex吗？
- notify\_once的once是唤醒哪一个呢？
- 什么时候会唤醒丢失？以及为什么会虚假唤醒呢？

(小朋友你为什么有这么多问号？)

### wait\_for和notify\_all的正常逻辑

wait_for之前lock the mutex，判断条件是否满足，满足直接解锁返回，不满足则解锁，将当前线程挂起并放入等待队列，直到超时或者被notify才会唤醒。

```c
# condition_variable
while (!__p())
	  if (wait_until(__lock, __atime) == cv_status::timeout)
	    return __p();
```

而notify\_all其实可以理解为等待队列中所有线程的逐个notify，毕竟存在共同的mutex，所以这里的mutex作用之一是保证线程唤醒、条件访问的原子性避免唤醒丢失，(另外还包括进入wait到添加到等待队列行为的原子性)。

> 在计算机科学中存在惊群问题，即当多个线程在等待同一个事件时，当事件发生后所有线程被唤醒，但只有一个线程可以被执行，其他线程又将被阻塞，进而造成计算机系统严重的上下文切换。

notify\_all并不是惊群行为，真实的实现并不是字面的“notify all”，实际是异步行为，在前一个线程出于critical area中时，其他线程并未被真正唤醒。原本一个线程需要分别等待notify和mutex，notify后发现没有mutex那么又阻塞，即形成了惊群。实际上，由linux内核里的futex\_requeue解决了该问题。

> static int futex_requeue(u32 user *uaddr1, unsigned int flags, u32 user *uaddr2, int nr_wake, int nr_requeue, u32 *cmpval, int requeue_pi);

从uaddr1中出队最多nr\_wake+nr\_requeue个futex\_q，并唤醒其中的nr\_wake个，其余进入uaddr2中等待。pthread_cond_broadcast中这两个size分别是1和INT_MAX，另外源码实现中是唤醒top waiter。

所以这个环节会导致预期的timeout会长一些，简单的条件逻辑大概会长百微秒级别(仅实验，实际取决于条件逻辑和硬件平台)。

### 同一condition variable wait\_for不同mutex，notify\_all会发生什么?

> Calling this function if `lock.mutex()` is not the same mutex as the one used by all other threads that are currently waiting on the same condition variable is undefined behavior.

都说同一个con\_var得绑定同一个mutex，不同的mutex则是undefined，所以什么是undefined以及mutex到底干了啥呢？

先说结果，只要第一个实现绑定的wait\_for可以立即唤醒，其他线程实现挂起，但无法唤醒直到timeout。

冷静分析：

POSIX说了：

> When a thread waits on a condition variable, having specified a particular mutex to either the `pthread_cond_timedwait()` or the `pthread_cond_wait()` operation, a **dynamic binding** is formed between that mutex and condition variable that remains in effect as long as at least one thread is blocked on the condition variable. During this time, the effect of an attempt by any thread to wait on that condition variable using a different mutex is undefined. Once all waiting threads have been unblocked (as by the `pthread_cond_broadcast()` operation), the next wait operation on that condition variable shall form a new dynamic binding with the mutex specified by that wait operation.

总的来说，con\_var和mutex之间存在动态绑定行为，关于mutex到底在保护啥后面在分析。对于上面提到的当等待的线程结束wait并成功解锁退出后，新的绑定行为可以重新建立。

### wait\_for之前加锁了，那么notify之前需要加锁吗？

wait\_for之前加锁一方面给wait\_for传参另一方面会去检测条件，加锁没毛病，那么notify之前需要加锁吗？

> The notifying thread does not need to hold the lock on the same mutex as the one held by the waiting thread(s); in fact doing so is a pessimization, since the notified thread would immediately block again, waiting for the notifying thread to release the lock.

For example:

```cpp
{
  std::lock_guard<std::mutex> lk(cv_m);
  std::cerr << "Notifying...\n";
}
cv.notify_all();
```

### notify_once真的是随机吗？

虽然多个wait_for之间使用了共同的mutex，但这只能保证互斥，那么notify_all后的线程唤醒顺序以及notify_once会唤醒哪一个线程呢？

网上搜到的答案都是说“是随机的，但是由具体实现决定，且程序设计中不要依赖顺序假设。”，只是实验结果来看表现为队列性质。这个问题直观上来说，我认为是有一个“队列”存在的，只是这个“队列”的生成规则有待研究。

> Notify one
> Unblocks one of the threads currently waiting for this condition.
> If no threads are waiting, the function does nothing.
> If more than one, it is unspecified which of the threads is selected.

> The effects of notify_one()/notify_all() and each of the three atomic parts of wait()/wait_for()/wait_until() (unlock+wait, wakeup, and lock) take place in a single total order that can be viewed as modification order of an atomic variable: the order is specific to this individual condition_variable.

从具体的实现来看，wait\_for实际在调用\_\_pthread\_cond\_timedwait，所以明显是顺序入队，且顺序完全取决调用顺序。

```c
# __pthread_cond_timedwait
node.next = c->_c_head;
		c->_c_head = &node;
		if (!c->_c_tail) c->_c_tail = &node;
		else node.next->prev = &node;
```

需要说明的是这里的c实际是内部自定义的condition variable，和用户使用的是两个东西。能想到的，不一定保证顺序执行是因为入队顺序不一定取决于用户接口调用的顺序，还需要考虑内部实现使用的condition variable和mutex，所以不能保证遵循用户的顺序性是合理的。

### 唤醒丢失和虚假唤醒

- 唤醒丢失

For example:

```cpp
# Process A                           # Process B

pthread_mutex_lock(&mutex);
while (condition == FALSE)
                                      condition = TRUE;
                                      pthread_cond_signal(&cond);
pthread_cond_wait(&cond, &mutex);
```

mutex的作用之一则可以保证用户条件的原子性。

- 虚假唤醒

> Even after a condition variable appears to have been signaled from a waiting thread's point of view, the condition that was awaited may still be false. One of the reasons for this is a spurious wakeup; that is, a thread might be awoken from its waiting state even though no thread signaled the condition variable. For correctness it is necessary, then, to verify that the condition is indeed true after the thread has finished waiting. Because spurious wakeup can happen repeatedly, this is achieved by waiting inside a loop that terminates when the condition is true, for example:
>
> ``` cpp
> /* In any waiting thread: */
> while (!buf->full)
> 	wait(&buf->cond, &buf->lock);
> 
> /* In any other thread: */
> if (buf->n >= buf->size) {
> 	buf->full = 1;
> 	signal(&buf->cond);
> }
> ```
>
> In this example it is known that another thread will set `buf->full` (the actual condition awaited) before signaling `buf->cond` (the means of synchronizing the two threads). The waiting thread will always verify the truth of the actual condition upon returning from `wait`, ensuring correct behaviour if spurious wakeup occurs.
>
> According to David R. Butenhof's Programming with POSIX Threads [ISBN](https://en.wikipedia.org/wiki/ISBN_(identifier)) [0-201-63392-2](https://en.wikipedia.org/wiki/Special:BookSources/0-201-63392-2):
>
> "This means that when you wait on a condition variable, the wait may (occasionally) return when no thread specifically broadcast or signaled that condition variable. Spurious wakeups may sound strange, but on some multiprocessor systems, making condition wakeup completely predictable might substantially slow all condition variable operations. The [race conditions](https://en.wikipedia.org/wiki/Race_condition) that cause spurious wakeups should be considered rare."

wiki大法好，以前确实也会困惑，使用wait的方式条件是用户管理的为什么还要extral while loop，实际如上“Spurious wakeups may sound strange, but on some multiprocessor systems, making condition wakeup completely predictable might substantially slow all condition variable operations.”。

### 总结

- condition variable和mutex是一一对应关系，如对应不同的mutex为undefined行为，表现为只有第一次绑定的con\_var-mutex对应的wait可以正常唤醒，其他wait无法唤醒只能timeout，或者在绑定关系结束后，重新绑定的线程可以唤醒；
- notify之前修改条件过程必须加锁，notify建议在解锁之后执行，避免wait线程唤醒后二次阻塞；
- notify\_once表现为顺序唤醒，只是顺序不仅取决于用户调用顺序还取决于api实现中自有的condition variable和mutex，所以程序开发不可依据顺序假设；
- 修改用户条件事先请加锁，避免唤醒丢失；
- wait api可能会存在虚假唤醒，需要配合用户条件逻辑使用。

### Ref

https://en.cppreference.com/w/cpp/thread/condition_variable

https://en.cppreference.com/w/cpp/thread/condition_variable/notify_one

https://en.cppreference.com/w/cpp/thread/condition_variable/notify_all

https://wiki.sei.cmu.edu/confluence/display/c/POS53-C.+Do+not+use+more+than+one+mutex+for+concurrent+waiting+operations+on+a+condition+variable

https://stackoverflow.com/questions/46461509/one-condition-variable-multiple-mutexes

https://www.zhihu.com/question/52757941

https://www.cnblogs.com/bbqzsl/p/6808176.html

https://stackoverflow.com/questions/4544234/calling-pthread-cond-signal-without-locking-mutex

https://elixir.bootlin.com/musl/latest/source/src/thread/pthread_cond_timedwait.c#L62

https://en.wikipedia.org/wiki/Spurious_wakeup