### 简易版限流算法实现

```java
package com.byterun.limit;

import java.util.Objects;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.LockSupport;

/**
 * 简陋版限流算法
 */
public class BuckLimitDemo {

    public static class BucketLimit {
        static AtomicInteger threadNum = new AtomicInteger(1);
        // 容量
        private int capcity;
        // 流速
        private int flowRate;
        // 流速时间单位
        private TimeUnit flowRateUnit;
        // 漏桶流出的事件间隔（纳秒）
        private long flowRateNanosTime;

        private BlockingQueue<Node> queue;

        public BucketLimit(int capcity, int flowRate, TimeUnit flowRateUnit) {
            this.capcity = capcity;
            this.flowRate = flowRate;
            this.flowRateUnit = flowRateUnit;
            this.bucketThreadWork();
        }

        // 构建一个桶
        public static BucketLimit build(int capcity, int flowRate, TimeUnit flowRateUnit) {
            if (capcity < 0 || flowRate < 0) {
                throw new IllegalArgumentException(("capcity/flowRate必须大于0"));
            }

            return new BucketLimit(capcity, flowRate, flowRateUnit);
        }

        // 漏桶线程
        public void bucketThreadWork () {
            this.queue = new ArrayBlockingQueue<>(capcity);
            // 漏桶流出的任务时间间隔（纳秒）
            this.flowRateNanosTime = flowRateUnit.toNanos(1) / flowRate;
            Thread thread = new Thread(this::buckWork);
            thread.setName("漏桶线程开始工作-" + threadNum.getAndIncrement());
            thread.start();
        }

        // 漏桶线程开始工作
        public  void buckWork() {
            while (true) {
                Node node = this.queue.poll();
                if (Objects.nonNull(node)) {
                    // 唤醒任务线程
                    LockSupport.unpark(node.thread);
                    System.out.println("唤醒任务线程=>" + node.thread.getName() +",剩余线程：" +this.queue.size());
                }
                // 休眠flowRateNanosTime
                LockSupport.parkNanos(this.flowRateNanosTime);
            }
        }

        // 漏桶申请
        public boolean acquire() {
            Thread currentThread = Thread.currentThread();
            Node node = new Node(currentThread);
            if (this.queue.offer(node)) { // 有效，进入漏桶
                LockSupport.park();  // 等待唤醒
                return true;
            }
            return false; // 漏桶满
        }


        // 漏桶中存放的元素
        class Node {
            private Thread thread;

            public Node(Thread thread) {
                this.thread = thread;
            }
        }
    }

    public static void main(String[] args) {
        BucketLimit bucketLimit = BucketLimit.build(10, 30, TimeUnit.MINUTES);
        for (int i = 0; i < 15; i++) {
            new Thread(() -> {
                boolean acquire = bucketLimit.acquire();
                System.out.println(System.currentTimeMillis() + "=>" + acquire);
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }

}

```

>  打印效果：

```powershell
1585747624268=>false
1585747624268=>false
1585747624268=>false
1585747624268=>false
1585747624268=>false
唤醒任务线程=>Thread-1,剩余线程：9
1585747626269=>true
唤醒任务线程=>Thread-4,剩余线程：8
1585747628274=>true
唤醒任务线程=>Thread-3,剩余线程：7
1585747630274=>true
唤醒任务线程=>Thread-2,剩余线程：6
1585747632276=>true
唤醒任务线程=>Thread-8,剩余线程：5
1585747634282=>true
唤醒任务线程=>Thread-9,剩余线程：4
1585747636282=>true
唤醒任务线程=>Thread-5,剩余线程：3
1585747638283=>true
唤醒任务线程=>Thread-10,剩余线程：2
1585747640288=>true
唤醒任务线程=>Thread-6,剩余线程：1
1585747642291=>true
唤醒任务线程=>Thread-7,剩余线程：0
1585747644295=>true

```

> 文献参考

* [BlockingQueue阻塞队列](https://mp.weixin.qq.com/s?__biz=MzA5MTkxMDQ4MQ==&mid=2648933190&idx=1&sn=916f539cb1e695948169a358549227d3&chksm=88621b78bf15926e0a94e50a43651dab0ceb14a1fb6b1d8b9b75e38c6d8ac908e31dd2131ded&token=1963100670&lang=zh_CN&scene=21#wechat_redirect)

* [JUC中的LockSupport工具类](https://mp.weixin.qq.com/s?__biz=MzA5MTkxMDQ4MQ==&mid=2648933125&idx=1&sn=382528aeb341727bafb02bb784ff3d4f&chksm=88621b3bbf15922d93bfba11d700724f1e59ef8a74f44adb7e131a4c3d1465f0dc539297f7f3&token=1338873010&lang=zh_CN&scene=21#wechat_redirect)