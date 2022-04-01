---
title: "Java互斥锁ReentrantLock实现原理"
date: 2020-01-07T11:43:39+08:00
description: "深入了解Java互斥锁的实现原理与相关锁概念"
Tags:
- JUC
- 锁
Categories:
- Java
---
了解**AQS**实现原理之后，再来分析**ReentrantLock**代码就非常简单了，在学习互斥锁之前很有必要搞清楚可重入锁、公平锁、非公平锁几个概念。
什么是可重入锁？线程成功获取锁之后，可以多次进入临界区访问资源，**ReentrantLock**就是一种可重入锁，其可重入的实现依赖于AQS的父类AOS，
当然JVM的**synchronized**锁也是可重入锁，锁大部分场景下应该设计成可重入模式，否则很容易发生死锁。如果**synchronized**不可重入，那么下面代码中的线
程执行main方法的时候将发生死锁，因为线程进入**getAndInc**方法时已经成功获取锁，当在**getAndInc**方法中调用另一个同步方法**inc**时由于
不可重入则产生死锁。

```Java
public class LockStudy {
    private int count;
    public static void main(String[] args) {
        LockStudy ls = new LockStudy();
        ls.getAndInc(10);
        System.out.println(ls.count);
    }
    public synchronized int getAndInc(int c) {
        int count = this.count;
        inc(c);
        return count;
    }
    public synchronized void inc(int c) {
        count = count + c;
    }
}
```

公平锁与非公平锁相对比较好理解，锁在公平模式下，请求获取锁的线程进入FIFO队列按顺序竞争，不允许线程插队。非公平锁模式下，线程不会严格按照请求顺序排队，
后来的线程可能比先到的线程更早获取锁，实现原理在后面详细介绍。

**ReentrantLock**互斥锁是由其内部的两个静态类实现的，分别是**FairSync**与**NonfairSync**，从这个类名可知其实现的功能对应的是公平锁与非公平锁。

![reentrantlock](/img/reentrantlock.png)

**ReentrantLock**提供了两个构造方法，一个是无参构造方法，另一个支持boolean类型参数，指定当前是否公平锁，**ReentrantLock**默认情况下非公平锁模式，完全由
静态内部类**Sync**子类实现。

```Java
public class ReentrantLock implements Lock, java.io.Serializable {

    private final Sync sync;

    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
	
	......
}
```

**ReentrantLock**提供了四个竞争锁的方法，适用于不同的应用场景，代码中应用了大量的模板模式。

```Java
/**
 * 根据是否公平锁实现方式有所不同，分别由FairSync/NonfairSync实现
 */
public void lock() {
	sync.lock();
}

/**
 * 根据是否公平锁实现方式有所不同，acquireInterruptibly是从AQS继承过来的方法，
 * 该方法再调用子类FairSync/NonfairSync实现的tryAcquire方法。
 * 支持中断，线程中断会抛出异常
 */
public void lockInterruptibly() throws InterruptedException {
	sync.acquireInterruptibly(1);
}

/**
 * 尝试快速获取锁，获取成功返回true，失败返回false，线程不会阻塞。
 * 该方法由Sync实现，是FairSync/NonfairSync共同逻辑，不管当前是否公平锁模式，都是强行获取锁，
 * 调用此方法，则使得ReentrantLock在公平模式下并非完全公平
 */
public boolean tryLock() {
	return sync.nonfairTryAcquire(1);
}

/**
 * 根据是否公平锁实现方式有所不同，tryAcquireNanos是从AQS继承过来的方法，
 * 该方法再调用子类FairSync/NonfairSync实现的tryAcquire方法。
 * 支持中断，线程中断会抛出异常
 */
public boolean tryLock(long timeout, TimeUnit unit)
		throws InterruptedException {
	return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

首先看**tryLock**的实现代码，即**Sync**的**nonfairTryAcquire**方法，Sync继承的是AQS，AQS中的状态state变量是32位的int类型，
理论上最大支持2^<sup>31</sup>-1=2147483647个线程并发竞争锁。

```Java
final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程对象
	final Thread current = Thread.currentThread();
	// 调用AQS的方法获取锁状态
	int c = getState();
	// 当前锁状态自由，未被任何线程占有，当前线程可以尝试获取
	if (c == 0) {
	    // 调用AQS的CAS方法修改状态，修改成功，再调用AOS方法记录当前线程，修改失败，表示其他线程已经获取了锁
		if (compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	// 状态不为0，表示已经有线程已经占用了锁，此时通过AOS的方法取出占用锁的线程，判断是不是当前线程，
	// 如果是当前线程，则直接将状态加1
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires;
		if (nextc < 0) // overflow
			throw new Error("Maximum lock count exceeded");
		// 这里无须用CAS，不会有线程竞争，未获取锁的线程不会执行到这儿
		setState(nextc);
		return true;
	}
	return false;
}
```

从上面代码可以看出**ReentractLock**是可以重入的，其可重入的实现关键是AOS，线程成功获取锁之后，释放锁之前会一直保存，线程重入一次，锁的状态加1。
下面再分析下**lock()**方法分别在公平与非公平模式下的实现，非公平模式实现相对比较简单，其效率也比公平锁高，尤其是在线程竞争比较激烈的场景下。

```Java
static final class NonfairSync extends Sync {
	private static final long serialVersionUID = 7316153563782823691L;

	final void lock() {
	    // 不管是否有线程在AQS的FIFO队列中排队等待，直接执行一次CAS操作竞争锁
		if (compareAndSetState(0, 1))
			setExclusiveOwnerThread(Thread.currentThread());
		else
		// CAS失败，则准备进入FIFO队列，在进入队列之前，还有一次机会，
		// AQS的acquire方法通过调用tryAcquire再给当前线程一次机会，此时再失败则进入队列等待
			acquire(1);
	}

	protected final boolean tryAcquire(int acquires) {
		return nonfairTryAcquire(acquires);
	}
}
```

从上面代码得知，在非公平模式下每个线程都有2次机会(CAS操作)插队竞争锁，2次均失败之后才会进入FIFO队列等待，然后公平锁模式下，线程是不允许插队竞争锁的，
只要FIFO队列中有线程在等待，则当前竞争锁的线程必须进入队列等待，这就是为什么公平锁的吞吐比非公平锁低的原因。

```Java
static final class FairSync extends Sync {
	private static final long serialVersionUID = -3000897897090466540L;

	// acquire方法会直接调用下面的tryAcquire方法，tryAcquire方法返回false，线程则进入队列等待
	final void lock() {
		acquire(1);
	}

	// 这个方法与Sync中nonfairTryAcquire方法逻辑不同点在于hasQueuedPredecessors
	protected final boolean tryAcquire(int acquires) {
		final Thread current = Thread.currentThread();
		int c = getState();
		if (c == 0) {
			if (!hasQueuedPredecessors() &&
				compareAndSetState(0, acquires)) {
				setExclusiveOwnerThread(current);
				return true;
			}
		}
		else if (current == getExclusiveOwnerThread()) {
			int nextc = c + acquires;
			if (nextc < 0)
				throw new Error("Maximum lock count exceeded");
			setState(nextc);
			return true;
		}
		return false;
	}
}
```

**hasQueuedPredecessors**是AQS提供的方法，用于判断当前队列中是否有线程在等待。

```Java
public final boolean hasQueuedPredecessors() {
	// 头与尾指向同一个节点那队列肯定是空
	// h.next == null 此时表示肯定有其他线程通过enq方法在进入队列，还未来的及设置next
	Node t = tail; // Read fields in reverse initialization order
	Node h = head;
	Node s;
	return h != t &&
		((s = h.next) == null || s.thread != Thread.currentThread());
}
```

不管是否公平锁，锁的释放逻辑一致，都是由**Sync**中的**tryRelease**方法实现，锁的释放调用顺序与锁的获取调用顺序刚好相反，
重入一次必须释放一次，最早调用最后释放。

```Java
protected final boolean tryRelease(int releases) {
	int c = getState() - releases;
	// 非持有锁的线程调用此方法直接抛出异常
	if (Thread.currentThread() != getExclusiveOwnerThread())
		throw new IllegalMonitorStateException();
	boolean free = false;
	// 状态为0，表示锁完全释放，此时需清除AOS中的线程记录
	if (c == 0) {
		free = true;
		setExclusiveOwnerThread(null);
	}
	setState(c);
	return free;
}
```
