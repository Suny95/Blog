title: 多线程笔记 (六) J.U.C之Condition
date: 2019-04-28 16:33:19
tags:  
- 线程
- condition
categories:
- 多线程
---
** {{ title }}：** <Excerpt in index | 首页摘要>
J.U.C之Condition
<!-- more -->
<The rest of contents | 余下全文>

本文参考地址：[【死磕Java并发】—–J.U.C之Condition](http://cmsblogs.com/?p=2222)

- Condition

> JUC提供的Condition是为了更加灵活的对wait、notify操作。使用Condition可以在未达到某个条件时阻塞线程，条件满足时又可以唤醒线程。

- Condition常用方法

> await()：使当前线程在被唤醒或中断之前一直处于等待状态，同时会加入到Condition等待队列同时释放锁。
>
> signal()：唤醒一个在等待队列中等待最长时间的节点（条件队列里的首节点）。该线程从等待方法返回前必须获得与Condition相关的锁。

**应用例子:**

```java
/**
 * 条件队列
 * @author sunny
 * @date 2019/3/6
 */
public class ConditionTest {
    public static void main(String[] args) {
        ConditionTest condition = new ConditionTest(2);

        ExecutorService pool = new ThreadPoolExecutor(2, 3,0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
        EnterCar p1 = new EnterCar(condition, "A车");
        EnterCar p2 = new EnterCar(condition, "B车");
        EnterCar p3 = new EnterCar(condition, "C车");

        LeaveCar c1 = new LeaveCar(condition);
        LeaveCar c2 = new LeaveCar(condition);
        LeaveCar c3 = new LeaveCar(condition);

        pool.execute(p1);
        pool.execute(p2);
        pool.execute(p3);
        pool.execute(c1);
        pool.execute(c2);
        pool.execute(c3);

        pool.shutdown();
    }

    /** 停车场 */
    private LinkedList<String> buffer;
    /** 停车场车位数 */
    private int maxSize;
    private Lock lock;
    /** 进入条件对象 */
    private Condition leaveCondition;
    /** 离开条件对象 */
    private Condition enterCondition;

    public ConditionTest(int maxSize) {
        this.maxSize = maxSize;
        this.buffer = new LinkedList<>();
        this.lock = new ReentrantLock();
        this.leaveCondition = lock.newCondition();
        this.enterCondition = lock.newCondition();
    }

    /**
     * 车辆进入停车场
     * 当停车场满了之后,进入条件对象等待.当有车辆离开时将会通知车辆可以进入.此时离开条件对象会通知有车进入
     *
     * @param str
     * @author sunny
     * @date 2019/4/28
     * @return void
     */
    public void enter(String str) throws InterruptedException {
        lock.lock();
        try {
            //当停车场满了之后就停止车辆进入
            while (maxSize == buffer.size()) {
                //满了,添加的线程进入等待状态
                System.out.println("车位已满---请等待");
                enterCondition.await();
            }

            buffer.add(str);
            System.out.println(str + " 停入停车场");
            leaveCondition.signal();
        }finally {
            lock.unlock();
        }
    }

    /**
     * 车辆离开停车场
     * 当停车场没有车时,离开条件对象等待.当有车辆进入时将会收到通知,此时可以开车离开停车场,进入条线对象会通知有车离开
     *
     * @author sunny
     * @date 2019/4/28
     * @return java.lang.String
     */
    public String leave() throws InterruptedException {
        String str;
        lock.lock();
        try {
            while (buffer.size() == 0) {
                System.out.println("停车场无车---");
                leaveCondition.await();
            }
            //返回停车场第一辆停的车
            str = buffer.poll()  + " 开出了停车场";
            //唤醒队列中第一个线程
            enterCondition.signal();
        } finally {
            lock.unlock();
        }
        return str;
    }
}

@Data
@AllArgsConstructor
class EnterCar extends Thread {
    private ConditionTest condition;
    private String str;

    @Override
    public void run() {
        try {
            condition.enter(str);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

@Data
@AllArgsConstructor
class LeaveCar extends Thread {
    private ConditionTest condition;

    @Override
    public void run() {
        try {
            System.out.println(condition.leave());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

