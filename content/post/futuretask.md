---
title: "FutureTask原理分析"
date: 2019-03-25T16:31:58+08:00
thumbnail: "img/futuretask.png"
description: "深入分析FutureTask原码，彻底了解FutureTask的执行原理"
Tags:
- Java源码
Categories:
- Java
---
### Callable Runnable
在Java中可以通过继承Thread或者实现Runnable接口两种方式来创建多线程，这两种方式创建的线程执行完毕之后，我们无法获取执行结果，
除非通过共享变量或者线程通信方式(Q消息等)间接实现，Java在1.5之后可以通过Callable和Future接口在线程执行完毕之后获取执行结果。
在了解FutureTask之前，有必要先了解清楚Java中Callable、Runnable之间的区别，Callable是java.util.concurrent包下的接口，而Runnable
是java.lang包下的接口。

```Java	
public interface Runnable {

	public abstract void run();
}
```
	
---

```Java
public interface Callable<V> {

	V call() throws Exception;
}
```
面试中经常会被问到Callable与Runnable的区别，至少要check到以下三个关键点。

- Runnable中的run方法是abstract方法
- Callable方法有返回值，Runnable的方法无返回值
- Callable的call方法允许抛异常，而Runnable的run不可抛异常

Runnable与Callable本身并没有任何关系，但是可以通过java.util.concurrent中Executors的callable方法将Runnable包装成Callable。

```Java
//包装成具有自定义返回值的Callable
public static <T> Callable<T> callable(Runnable task, T result) {
	if (task == null)
		throw new NullPointerException();
	return new RunnableAdapter<T>(task, result);
}
//包装成不需要返回值的Callable
public static Callable<Object> callable(Runnable task) {
	if (task == null)
		throw new NullPointerException();
	return new RunnableAdapter<Object>(task, null);
}
//通过静态内部类RunnableAdapter实现
static final class RunnableAdapter<T> implements Callable<T> {
	final Runnable task;
	final T result;
	RunnableAdapter(Runnable task, T result) {
		this.task = task;
		this.result = result;
	}
	public T call() {
		task.run();
		return result;
	}
}
```
### FutureTask父接口
FutureTask的继承关系

![futuretask](/blog/futuretask/001.png)

从上图中看出，FutureTask实现了RunnableFuture接口，RunnableFuture接口又继承了Future、Runnable接口，所以FutureTask既可以作为一个Task
被其他的Thread或者ThreadPool执行，又可以作为Future用来获取Callable的执行结果。Future表示的是异步计算结果，该接口定义了5个核心概念方法。

- **boolean cancel(boolean mayInterruptIfRunning)**<br/>
	该方法用来试着取消任务的执行，方法返回值代表了取消操作是否成功，任务已经完成、已经被取消或者由于某些原因不能取消则操作失败，<br/>
	任务还未开始执行，则取消成功，如果任务已经开始执行，参数mayInterruptIfRunning决定了执行任务线程是否应该被中断。<br/>
	`注意:任务开始执行，mayInterruptIfRunning为true，中断了执行任务的线程，仅仅是设置了该线程的中断标志，至于任务是否真正的停止执行具体要看任务逻辑的实现。`
	```Java
		public class MyTask implements Runnable {

			@Override
			public void run() {
				if (Thread.interrupted()) {
					System.out.println("Thread interrupted, stop execute task");
				} else {
					System.out.println("do business");
				}
			}
		}
	```
	
- **boolean isCancelled()**<br/>
	返回任务是否在执行完成之前被取消。
- **boolean isDone()**<br/>
	返回任务是否已经完成，包括正常执行结束、异常或者取消，只要是终结状态则返回true。
- **V get() throws InterruptedException, ExecutionException;**<br/>
	返回任务执行结果，如果任务正在执行，调用get方法的线程则被阻塞，阻塞过程中被中断，则抛InterruptedException，<br/>
	如果线程被取消抛运行时异常CancellationException，如果任务执行发生异常抛ExecutionException。
- **V get(long timeout, TimeUnit unit)**<br/>
	该方法多了一个等待时长，在给定的时间范围内任务还未执行完毕，则抛出TimeoutException。

RunnableFuture接口中未增加新的方法，重新申明了父类Runnable中的run方法。

```Java
public interface RunnableFuture<V> extends Runnable, Future<V> {

    void run();
}
```

### FutureTask

FutureTask中的任务类型是Callable，整个过程中这个Callable都具有一个状态state，首先看看FutureTask中定义的一些变量。

```Java
//Callable任务的状态，可能有下面的0-6种状态值
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;

//用来存放传入的任务
private Callable<V> callable;
//用来保存任务的成功执行的结果后者执行任务时发生异常的Exception对象
private Object outcome; 
//用来存放执行Callable任务的线程
private volatile Thread runner;
//调用get方法获取任务执行结果时被阻塞的线程栈
private volatile WaitNode waiters;
```

WaitNode的next属性引用了下一个WaitNode，这样构成了一个链表结构。

```Java
static final class WaitNode {
	volatile Thread thread;
	volatile WaitNode next;
	WaitNode() { thread = Thread.currentThread(); }
}
```

用volatile修饰的变量保证了线程间的可见性，在FutureTask中这些变量都是通过CAS原子操作。

```Java
private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset;
private static final long runnerOffset;
private static final long waitersOffset;
static {
	try {
		UNSAFE = sun.misc.Unsafe.getUnsafe();
		Class<?> k = FutureTask.class;
		stateOffset = UNSAFE.objectFieldOffset
			(k.getDeclaredField("state"));
		runnerOffset = UNSAFE.objectFieldOffset
			(k.getDeclaredField("runner"));
		waitersOffset = UNSAFE.objectFieldOffset
			(k.getDeclaredField("waiters"));
	} catch (Exception e) {
		throw new Error(e);
	}
}
```

#### 任务状态
FutureTask中callable的状态一共7种，初始状态NEW，中间状态COMPLETING、INTERRUPTING，最终状态NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTED，
中间状态一般是瞬时的，很快会过渡到最终状态。

![futuretask](/blog/futuretask/002.png)

#### 构造方法
FutureTask是Java中唯一实现了RunnableFuture接口的类，它提供了两个构造方法，分别支持以Callable、Runnable类型传入任务，任务类型是
Runnable的则利用Executors.callable方法将Runnable包装Callable，上面已经展示相关包装代码。FutureTask构造完成之后，其任务callable的状态为初始状态NEW。

```Java
public FutureTask(Callable<V> callable) {
	if (callable == null)
		throw new NullPointerException();
	this.callable = callable;
	this.state = NEW;       // ensure visibility of callable
}


public FutureTask(Runnable runnable, V result) {
	this.callable = Executors.callable(runnable, result);
	this.state = NEW;       // ensure visibility of callable
}
```

#### run方法
FutureTask实现了Runnable接口，因此run方法是执行线程或者线程池调度执行核心方法，详细分析run方法源码。

```Java
public void run() {
	//Thread.currentThread此时是任务执行线程，不是发起线程
	if (state != NEW ||
		!UNSAFE.compareAndSwapObject(this, runnerOffset,
									 null, Thread.currentThread()))
		return;
	try {
		Callable<V> c = callable;
		if (c != null && state == NEW) {
			V result;
			boolean ran;
			try {
				result = c.call();
				ran = true;
			} catch (Throwable ex) {
				result = null;
				ran = false;
				setException(ex);
			}
			if (ran)
				set(result);
		}
	} finally {
		// runner must be non-null until state is settled to
		// prevent concurrent calls to run()
		runner = null;
		// state must be re-read after nulling runner to prevent
		// leaked interrupts
		int s = state;
		if (s >= INTERRUPTING)
			handlePossibleCancellationInterrupt(s);
	}
}
```

- 判断任务状态不是NEW，表示任务已经执行过，直接返回
- CAS设置执行线程runner，设置失败表示已经有线程在执行该任务，直接返回
- 再次判断任务状态是否是初始状态，并且callable是否为null
- 满足条件之后直接调用callable的call方法，实际调用了Runnable的run方法
- 任务执行成功调用set(result)设置返回结果，任务执行发生异常，调用setException(ex)设置异常对象信息
- finally中语句重新获取状态state，因为有可能在执行run方法的时候，发起线程调用了FutureTask的cancel(true)方法中断执行线程，<br/>
	此时state的状态有可能为EXCEPTIONAL、NORMAL、INTERRUPTING、INTERRUPTED任何一种
- 如果state状态此时为INTERRUPTING，表示有线程正在取消(中断)任务，此时执行线程执行Thread.yield()让出cpu执行时间，尽快让发起线程将state设置为最终状态INTERRUPTED

```Java
private void handlePossibleCancellationInterrupt(int s) {
	if (s == INTERRUPTING)
		while (state == INTERRUPTING)
			Thread.yield(); 
    }
```

任务正常执行结束，将执行结果赋值给变量outcome，state从COMPLETING状态过渡到NORMAL状态。

```Java
protected void set(V v) {
	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
		outcome = v;
		UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
		finishCompletion();
	}
}
```
任务执行过程中发生异常，将异常对象赋值给变量outcome，因为outcome是一个Object类型，state从COMPLETING过渡到EXCEPTIONAL。

```Java
protected void setException(Throwable t) {
	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
		outcome = t;
		UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
		finishCompletion();
	}
}
```

不管任务正常执行结束，还是发生异常，最后都要调用finishCompletion方法处理，该方法主要用来唤醒因调用get方法而阻塞的线程，同时
将FutureTask中的一些变量比如callable赋值null，最后调用done()方法，done方法在FutureTask中空实现，子类可以进行覆盖。

```Java
private void finishCompletion() {
	// 通过for循环调用CAS操作，保证操作一定可以成功
	for (WaitNode q; (q = waiters) != null;) {
		if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
			for (;;) {
			    //取出节点中的线程对象
				Thread t = q.thread;
				if (t != null) {
					q.thread = null;
					//唤醒线程
					LockSupport.unpark(t);
				}
				//处理下一个等待节点
				WaitNode next = q.next;
				//next节点为空，表示所有的等待节点均已处理完毕，退出循环
				if (next == null)
					break;
				q.next = null; // unlink to help gc
				q = next;
			}
			break;
		}
	}
	//子类可以覆盖该方法，增加一些额外的业务逻辑
	done();

	callable = null;        
}
```

FutureTask中同时提供了方法isDone方法，用来表示任务是否处理完成，只要state的值不是初始状态NEW，返回true。

```Java
public boolean isDone() {
	return state != NEW;
}
```

#### cancel方法
cancel方法主要用来取消任务的执行，前面已经介绍过，并不是任务都能取消成功，同时要理解取消操作成功，也不能意味着任务就一定未执行，因为执行线程可能还未将state从NEW设置为COMPLETING，但是已经开始执行了。
取消方法带着一个参数mayInterruptIfRunning，该参数表示是否中断正在执行的任务，也就是说任务正在被某个执行线程处理，只是还未处理完成，针对这种情况根据该参数判断是否需要中断执行线程。

```Java
public boolean cancel(boolean mayInterruptIfRunning) {
	//任务状态不是初始状态NEW，则取消失败，直接返回false
	//或者CAS设置任务状态为INTERRUPTING/CANCELLED时失败，表示状态已经被某个执行线程修改了，直接返回false
	if (!(state == NEW &&
		  UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
			  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
		return false;
	try {
		//如果进入if语句，此时state状态肯定已经设置为INTERRUPTING
		if (mayInterruptIfRunning) {
			try {
				Thread t = runner;
				//中断任务的执行线程
				if (t != null)
					t.interrupt();
			} finally { 
				//将state状态从INTERRUPTING过渡到最终状态INTERRUPTED
				UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
			}
		}
	} finally {
		finishCompletion();
	}
	return true;
}
```
从cancel方法代码看出，如果参数mayInterruptIfRunning为false，取消操作很简单，通过cas操作将state从NEW设置为CANCELLED，即操作成功。
如果mayInterruptIfRunning为true，通过CAS操作将state从NEW设置为INTERRUPTING，最后变成INTERRUPTED。任务中断或者取消之后，同样调用
finishCompletion方法唤醒所有阻塞的线程。FutureTask中提供了方法isCancel判断任务是否已经被取消，它的逻辑则是只要state是CALCELLED、
INTERRUPTING、INTERRUPTED任何一种则表示任务被取消。

```Java
public boolean isCancelled() {
	return state >= CANCELLED;
}
```

#### get方法
FutureTask中提供了2个get方法，其中一个get方法不带参数，另外一个get方法有时间参数，表示获取任务执行结果时的最长等待时间，
超过给定的时间还未获取到执行结果，则抛出TimeoutException，两个get方法均支持线程中断，发成中断时抛出InterruptedException，
如果任务被取消，两个方法都会抛出运行时异常CancellationException。

```Java
public V get() throws InterruptedException, ExecutionException {
	int s = state;
	if (s <= COMPLETING)
		s = awaitDone(false, 0L);
	return report(s);
}

public V get(long timeout, TimeUnit unit)
	throws InterruptedException, ExecutionException, TimeoutException {
	if (unit == null)
		throw new NullPointerException();
	int s = state;
	if (s <= COMPLETING &&
		(s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
		throw new TimeoutException();
	return report(s);
}
```

两个get方法底层都是调用相同的awaitDone方法实现，只是传入参数区别，awaitDone方法是实现线程阻塞的核心方法。

```Java
private int awaitDone(boolean timed, long nanos)
	throws InterruptedException {
	//如果设置超时等待，计算出线程阻塞截止时间
	final long deadline = timed ? System.nanoTime() + nanos : 0L;
	WaitNode q = null;
	boolean queued = false;
	//线程开始自旋
	for (;;) {
		//如果自旋过程中，线程被中断，则将该线程从等待线程列表中移除，同时抛出InterruptedException
		if (Thread.interrupted()) {
			removeWaiter(q);
			throw new InterruptedException();
		}
		//每次获取state最新值
		int s = state;
		//如果state是最终状态，则退出自旋返回
		if (s > COMPLETING) {
			if (q != null)
				q.thread = null;
			return s;
		}
		else if (s == COMPLETING) // cannot time out yet
			//state是中间状态，当前线程让出cpu执行时间，让其他线程尽快执行将state设置成最终状态
			Thread.yield();
		else if (q == null)
			q = new WaitNode();
		else if (!queued)
			//waiters从这看出是一个链表形成的Stack，新加入的等待节点总是头节点
			queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
												 q.next = waiters, q);
		else if (timed) {
			nanos = deadline - System.nanoTime();
			//等待截止时间到了，则从等待节点栈中移除该线程节点，返回
			if (nanos <= 0L) {
				removeWaiter(q);
				return state;
			}
			//将线程阻塞指定时间
			LockSupport.parkNanos(this, nanos);
		}
		else
			//无时间限制，一直阻塞当前线程，直到其他线程调用finishCompletion唤醒
			LockSupport.park(this);
	}
}
```
### Demo

```Java
public class Test {

    public static void main(String[] args) {
        Callable<String> callable = () -> {
            System.out.println("doing");
            try {
                //mock do logic
                Thread.sleep(3000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println("do over");
            return "ok";
        };

        FutureTask<String> task = new FutureTask<>(callable);

        new Thread(task).start();

        try {
            System.out.println(System.currentTimeMillis());
            task.cancel(true);
            String result = task.get();
            System.out.println(System.currentTimeMillis());
            System.out.println("result: " + result);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

