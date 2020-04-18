# 线程状态说明

Tag: 线程 状态

线程状态基于Thread类中的 `State` 枚举

```java
/**
     * 线程状态  线程必须是下面状态中的一个
     * <ul>
     * <li>{@link #NEW}<br>
     *     线程还没启动的时候处于此状态
     *     </li>
     * <li>{@link #RUNNABLE}<br>
     *     在Java虚拟机中执行的线程处于这种状态。
     *     </li>
     * <li>{@link #BLOCKED}<br>
     *     在等待监视器锁时被阻塞的线程处于这种状态。
     *     </li>
     * <li>{@link #WAITING}<br>
     *     无限期等待另一个线程执行特定操作的线程处于这种状态。
     *     </li>
     * <li>{@link #TIMED_WAITING}<br>
     *     在指定等待时间内等待另一个线程执行某个操作的线程处于这种状态。
     *     </li>
     * <li>{@link #TERMINATED}<br>
     *     退出的线程处于这种状态。
     *     </li>
     * </ul>
     *
     * <p>
     * 在给定的时间点上，线程只能处于一种状态。
     * 这些状态是虚拟机状态，不反映任何操作系统线程状态。
     *
     * @since   1.5
     * @see #getState
     */
    public enum State {
        /**
         * 尚未启动的线程的线程状态。
         */
        NEW,

        /**
         * 可运行线程的线程状态。
         * 
         * 处于可运行状态的线程正在Java虚拟机中执行，但它可能正在等待来自操作系统(如处理器)的其他资源。
         */
        RUNNABLE,

        /**
         * 等待监视器锁的阻塞线程的线程状态。
         * 
         * 处于阻塞状态的线程正在等待一个监视器锁进入一个同步 块/方法，
         * 或者在调用{@link Object#wait() Object.wait}后重新进入一个同步 块/方法。
         */
        BLOCKED,

        /**
         * 等待中线程的线程状态。
         * 一个线程由于调用下列方法之一而处于等待状态:
         * <ul>
         *   <li>{@link Object#wait() Object.wait}未指定超时时间timeout</li>
         *   <li>{@link #join() Thread.join}未指定超时时间timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>处于等待状态的线程正在等待另一个线程执行特定的操作。
         *
         * 例如，一个线程在一个对象上调用了<tt>Object.wait()</tt>，
         * 正在等待另一个线程在该对象上调用<tt>Object.notify()</tt>或
         * <tt>Object.notifyAll()</tt>。
         * 调用<tt>Thread.join()</tt>的线程正在等待指定的线程终止。
         */
        WAITING,

        /**
         * 具有指定等待时间的等待线程的线程状态。
         * 线程处于定时等待状态，因为调用以下方法之一与指定的等待时间:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait}指定超时时间timeout</li>
         *   <li>{@link #join(long) Thread.join}指定超时时间timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * 终止线程的线程状态。线程已经完成执行。
         */
        TERMINATED;
    }
```

