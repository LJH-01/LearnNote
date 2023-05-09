[TOC]



# CompleteFuture之get()源码

## CompletableFuture结构

见CompletableFuture之complete()源码

## CompletableFuture.supplyAsync方法
该方法需要传入一个Supplier类型的函数接口，该函数接口是包含返回结果的。
1、 实际调用 asyncSupplyStage(asyncPool, supplier) 方法，带参一个线程池和一个Supplier类型的函数接口。

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
}
 static <U> CompletableFuture<U> asyncSupplyStage(Executor e,
                                                     Supplier<U> f) {
        if (f == null) throw new NullPointerException();
        CompletableFuture<U> d = new CompletableFuture<U>();
        // 这里比较重要的是理解 d 是引用类型的。execute 执行的是AsyncSupply中的run方法。
        e.execute(new AsyncSupply<U>(d, f));
        return d;
    }
static final class AsyncSupply<T> extends ForkJoinTask<Void>
            implements Runnable, AsynchronousCompletionTask {
        CompletableFuture<T> dep; Supplier<T> fn;
        //构造赋值
        AsyncSupply(CompletableFuture<T> dep, Supplier<T> fn) {
            this.dep = dep; this.fn = fn;
        }
  public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) {}
    public final boolean exec() { run(); return true; }

    public void run() {
        CompletableFuture<T> d; Supplier<T> f;
        if ((d = dep) != null && (f = fn) != null) {
            dep = null; fn = null;
            if (d.result == null) {
                try {
                	/**
                	CAS 进行赋值，赋值对象就是 result  
                	这个地方，可能有些情况不能及时获取到值，需要保证 result属性可见性
					volatile Object result; 
                	*/
                    d.completeValue(f.get());
                } catch (Throwable ex) {
                    d.completeThrowable(ex);
                }
            }
            // 完成后的相关处理，其中唤醒操作就在其中
            d.postComplete();
        }
    }
}
```


刚刚的分析其实不难，相信到这大概明白 CompletableFuture 是如何通过get方法获取返回结果的，
根据上面的源码可知，返回的就是创建的 CompletableFuture对象d，当调用CompletableFuture中的get方法获取的返回结果就是CompletableFuture中的result属性结果，它是通过CAS进行赋值的。

## CompletableFuture.get方法 

看代码和注释大概清楚是如何阻塞线程

```java
 public T get() throws InterruptedException, ExecutionException {
        Object r;
        /* 
        分为两种，线程执行比较快或者慢。执行比较慢的就进入 waitingGet(true)方法进行阻塞。
        快的就直接返回结果信息
        */
        return reportGet((r = result) == null ? waitingGet(true) : r);
    }
 //这个返回主要判断这个result是什么类型进行相关处理
 private static <T> T reportGet(Object r)
        throws InterruptedException, ExecutionException {
        if (r == null) // by convention below, null means interrupted
            throw new InterruptedException();
        if (r instanceof AltResult) {
            Throwable x, cause;
            if ((x = ((AltResult)r).ex) == null)
                return null;
            if (x instanceof CancellationException)
                throw (CancellationException)x;
            if ((x instanceof CompletionException) &&
                (cause = x.getCause()) != null)
                x = cause;
            throw new ExecutionException(x);
        }
        @SuppressWarnings("unchecked") T t = (T) r;
        return t;
    }
    
 private Object waitingGet(boolean interruptible) {
        Signaller q = null;
        boolean queued = false;
        int spins = -1;
        Object r;
        while ((r = result) == null) {
            if (spins < 0)
                //获取当前处理数量是否大于1，大于1 spins = 256 否则 0
                spins = (Runtime.getRuntime().availableProcessors() > 1) ?
                    1 << 8 : 0; // Use brief spin-wait on multiprocessors
            else if (spins > 0) {
                //获取一个随机的数值，如果每次都大于等于0，--spins 到小于0为止
                if (ThreadLocalRandom.nextSecondarySeed() >= 0)
                    --spins;
            }
            else if (q == null)
               /**
                 注意，这个对象很关键。用于放入到ForkjoinPool中，利用ForkjoinPool来处理，利用这个对象结合ForkjoinPool来控制线程的休眠与唤醒
		         Signaller(boolean interruptible, long nanos, long deadline) {
		            this.thread = Thread.currentThread();
		            this.interruptControl = interruptible ? 1 : 0;
		            this.nanos = nanos;
		            this.deadline = deadline;
		        }
               */
                q = new Signaller(interruptible, 0L, 0L);
            else if (!queued)
                queued = tryPushStack(q);
            else if (interruptible && q.interruptControl < 0) {
                q.thread = null;
                cleanStack();
                return null;
            }
            else if (q.thread != null && result == null) {
                try {
                	// 线程不为空，并且没有返回结果。就是线程执行慢要进行阻塞了
                    ForkJoinPool.managedBlock(q);
                } catch (InterruptedException ie) {
                    q.interruptControl = -1;
                }
            }
        }
        if (q != null) {
            q.thread = null;
            if (q.interruptControl < 0) {
                if (interruptible)
                    r = null; // report interruption
                else
                    Thread.currentThread().interrupt();
            }
        }
        postComplete();
        return r;
    }

public static void managedBlock(ManagedBlocker blocker)
        throws InterruptedException {
        ForkJoinPool p;
        ForkJoinWorkerThread wt;
        Thread t = Thread.currentThread();
        //当前测试线程是Main，直接分析else
        if ((t instanceof ForkJoinWorkerThread) &&
            (p = (wt = (ForkJoinWorkerThread)t).pool) != null) {
        }
        else {
            //当前blocker就是上面传参q，直接分析q类型里面的这些方法
            do {} while (!blocker.isReleasable() &&
                         !blocker.block());
        }
    }
    
public boolean block() {
		//false
       if (isReleasable())
           return true;
        //构造赋值时为0.
       else if (deadline == 0L)
           /**
           this对象就是 q，阻塞的就是当前线程。
           到这就找到了如何阻塞线程，那么他是如何定位该线程并唤醒的呢？
           */
           LockSupport.park(this);
       else if (nanos > 0L)
           LockSupport.parkNanos(this, nanos);
       return isReleasable();
   }

public boolean isReleasable() {
        if (thread == null)
            return true;
            //是否被中断
        if (Thread.interrupted()) {
            int i = interruptControl;
            interruptControl = -1;
            if (i > 0)
                return true;
        }
        //构造时，有赋值。已当前分析，不进入
        if (deadline != 0L &&
            (nanos <= 0L || (nanos = deadline - System.nanoTime()) <= 0L)) {
            thread = null;
            return true;
        }
        return false;
    }
```


接下来我们需要分析下它是如何或者说在哪调用 LockSupport.unPark方法在获取到执行结果的情况下进行唤醒操作的呢？此时，还记得执行AsyncSupply.run方法吗，最后它有一步完成操作，就是CompletableFuture.postComplete方法。

``` java
CompletableFuture.postComplete方法
 final void postComplete() {
        // this就是创建的对象d 
        CompletableFuture<?> f = this; Completion h;
        /**
        h = f.stack 到这我们需要分析下，当前是有两个线程Main和线程池里面的线程在跑不同的代码。
        Main线程将d（CompletableFuture） 里面的stack 赋值变成了q （Signaller），
        那么它如何保证线程间的通信？是不是只要属性使用 volatile属性修饰就可以保证可见性呀
        volatile Completion stack; 
    */
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        // CAS 将stack变成q的next 也就是null
        if (f.casStack(h, t = h.next)) {
            //不进入
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                h.next = null;    // detach
            }
            // h 是q （Signaller） NESTED = -1
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}

final CompletableFuture<?> tryFire(int ignore) {
            Thread w; // no need to atomically claim
            // thread就是Main线程，
            if ((w = thread) != null) {
                thread = null;
                //唤醒处理
                LockSupport.unpark(w);
            }
            return null;
        }
```

到这CompletableFuture是如何获取值，当执行结果返回比较慢是如何处理的相信大家有了更深一步的了解。

如果是有超时时间的get，则唤醒之后没有获取到result之后，则会抛出异常new TimeoutException();

有超时时间的get，见下

```java
private Object timedGet(long nanos) throws TimeoutException {
    if (Thread.interrupted())
        return null;
    if (nanos <= 0L)
        throw new TimeoutException();
    long d = System.nanoTime() + nanos;
    Signaller q = new Signaller(true, nanos, d == 0L ? 1L : d); // avoid 0
    boolean queued = false;
    Object r;
    // We intentionally don't spin here (as waitingGet does) because
    // the call to nanoTime() above acts much like a spin.
    while ((r = result) == null) {
        if (!queued)
            queued = tryPushStack(q);
        else if (q.interruptControl < 0 || q.nanos <= 0L) {
            q.thread = null;
            cleanStack();
            if (q.interruptControl < 0)
                return null;
            throw new TimeoutException();
        }
        else if (q.thread != null && result == null) {
            try {
                ForkJoinPool.managedBlock(q);
            } catch (InterruptedException ie) {
                q.interruptControl = -1;
            }
        }
    }
    if (q.interruptControl < 0)
        r = null;
    q.thread = null;
    postComplete();
    return r;
}
```

## 总结

1. supplyAsync方法将要执行的操作加入到ForkJoinPool中，异步执行。
2. get方法，在ForkJoinPool中加入Signaller，利用Signaller来控制线程的休眠以及唤醒

## 参考

https://blog.csdn.net/ljp7272/article/details/122997737
