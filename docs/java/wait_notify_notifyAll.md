# Java中wait/notify/notifyAll使用详解

> 发布于 2021-01-19

### 基础知识

这几个都是Object对象的方法, 即java中所有的对象都有这些方法, 且被申明为native方法, 主要用于线程间通信.

```java
public final native void notify();
public final native void notifyAll();
public final native void wait(long timeout) throws InterruptedException;
public final void wait() throws InterruptedException { wait(0); };
```

### 用法

Java语法中规定，当前线程必须获得对象锁, 才能调用此三个方法, java中使用synchronized关键字来获得对象锁, 在synchronized代码块或者方法中，必定是会持有对象锁,每个对象都可以作为synchronized锁的对象，因此wait、notify等必须和对象关联才能配合synchronized使用

```java
//作用于代码块
synchronized(object) {
    while(contidion) {
        object.wait();
    }
    //object.notify();
    //object.notifyAll();
}

//作用于方法
public synchronized void methodName() {
    while(contidion) {
        object.wait();
    }
    //object.notify();
    //object.notifyAll();
}
```

### 作用

| 方法  | 作用 |
| ---- | ---- |
| [wait](https://xxx) | 线程自动释放占有的对象锁, 并等待notify. |
| [notify](https://xxx) | 随机唤醒一个正在wait当前对象的线程, 并让被唤醒的线程拿到对象锁. |
| [notifyAll](https://xxx) | 唤醒所有正在wait当前对象的线程, 但是被唤醒的线程会再次去竞争对象锁. 因为一次只有一个线程能拿到锁, 所有其他没有拿到锁的线程会被阻塞. 推荐使用. |

### 实例

```java
import java.util.Queue;
import java.util.LinkedList;

public class TestDemo {
    private static Boolean canRun = true;
    private static final Integer MAX_CAPACITY = 7;
    private static final Queue<Double> queue = new LinkedList<Double>();

    public static void main(String[] args) {
        TestDemo demo = new TestDemo();
        //work b
        WorkerB b = demo.new WorkerB();
        b.start();
        //work a
        WorkerA a = demo.new WorkerA();
        a.start();
    }

    class WorkerA extends Thread {
        @Override
        public void run() {
            while (canRun) {
                synchronized (queue) {
                    while (queue.size() >= MAX_CAPACITY) {
                        try {
                            System.out.println("full");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    try {
                        double d = Math.random();
                        queue.add(d);
                        System.out.println("push:" + d);
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    queue.notifyAll();
                }
            }
        }
    }

    class WorkerB extends Thread {
        @Override
        public void run() {
            while (canRun) {
                synchronized (queue) {
                    while (queue.isEmpty()) {
                        try {
                            System.out.println("empty");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    try {
                        System.out.println("pop:" + queue.poll());
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    queue.notifyAll();
                }
            }
        }
    }
}

```

!!! 总结

    1. work A 和 work B 都有两层while,外层的while是用来判断是否可以执行.内层的while用来判断队列queue是否已满或者为空,如果满足条件,则使得当前线程变成等待状态(等待notify).
    2. 内层的条件判断为什么用while不用if,原因是当线程被wait之后,会释放对象锁.当等待的线程被notify之后,必须再次尝试去获取对象锁,如果没有获取到对象锁,那还必须等待,直到拿到对象锁之后才能向后执行.
    3. 当work A push了一个数据或者work B pop了一个数据之后,使用notifyAll()方法来通知所有等待当前对象锁的线程,但是一次只会有一个等待的线程能拿到锁, 到底谁运行不确定.
    4. 我们使用queue作为锁的对象在不同线程之间进行通信.
