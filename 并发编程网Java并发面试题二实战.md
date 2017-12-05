# 并发容器和框架

## 1. 如何让一段程序并发的执行，并最终汇总结果？
1.使用闭锁CountDownLatch：每个线程的任务执行结束后执行countdDown，在主线程中进行结果汇总。

2.使用循环栅栏CyclicBarrier：每个线程任务执行后在通过await等待所有线程执行结束后再继续执行结果汇总。

3.使用Future+线程池，任务提交给线程池后获得Future对象，在主线程中通过Future.get获取并汇总执行结果。

可以考虑直接使用CompeleteServcie进行优化，可以优先获取到已经执行完线程的结果。

## 2. 如何合理的配置java线程池？
如CPU密集型的任务，基本线程池应该配置多大？IO密集型的任务，基本线程池应该配置多大？
用有界队列好还是无界队列好？任务非常多的时候，使用什么阻塞队列能获取最好的吞吐量？
需要理解掌握线程池的配置属性，如：核心线程数-corePoolSize,最大线程数-maximumPoolSize,任务队列-runnableTaskQueue,饱和拒绝策略-RejectedExecutionHandler
需要理解分析任务的特性：CPU密集型、IO密集型、混合型，任务优先级，任务执行时间长短，任务对外部资源依赖。
CPU密集型应尽可能少的分配线程数，一般为N<sub>cpu</sub>+1；IO密集型任务因需要等待IO操作，可以多配置线程数，
一般为2*N<sub>cpu</sub>;混合型任务需要分析是否可以拆分为CPU密集型和IO密集型（执行时间差不多）。
`Runtime.getRuntime().availableProcessors()`可以获取当前机器CPU数。
使用有界队列好，有界队列可以增加系统稳定性和预警能力，可以根据需要设置稍大一点。使用有界队列可以更早的发现并发
中的问题，并且可以控制问题的影响范围。
以下引用[Java线程池的分析和使用](http://ifeve.com/java-threadpool/)：
> 要想合理的配置线程池，就必须首先分析任务特性，可以从以下几个角度来进行分析：
> 1. 任务的性质：CPU密集型任务，IO密集型任务和混合型任务。
> 2. 任务的优先级：高，中和低。
> 3. 任务的执行时间：长，中和短。
> 4. 任务的依赖性：是否依赖其他系统资源，如数据库连接。
> 5. 任务性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务配置尽可能少的线程数量，如配置Ncpu+1个线程的线程池。IO密集型任务则由于需要等待IO操作，线程并不是一直在执行任务，则配置尽可能多的线程，如2*Ncpu。混合型的任务，如果可以拆分，则将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐率要高于串行执行的吞吐率，如果这两个任务执行时间相差太大，则没必要进行分解。我们可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。
优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先得到执行，需要注意的是如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。
执行时间不同的任务可以交给不同规模的线程池来处理，或者也可以使用优先级队列，让执行时间短的任务先执行。
依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，如果等待的时间越长CPU空闲时间就越长，那么线程数应该设置越大，这样才能更好的利用CPU。建议使用有界队列，有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点，比如几千。有一次我们组使用的后台任务线程池的队列和线程池全满了，不断的抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞住，任务积压在线程池里。如果当时我们设置成无界队列，线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然我们的系统所有的任务是用的单独的服务器部署的，而我们使用不同规模的线程池跑不同类型的任务，但是出现这样问题时也会影响到其他任务。

## 3. 如何使用阻塞队列实现一个生产者和消费者模型？请写代码。
整体思路：定义生产者任务和消费者任务(Runnable)，定义阻塞队列。生产者任务产生待处理数据入队列，队列满则阻塞等待；消费者
从队列取待处理数据，队列空则阻塞等待。

以下示例根据原地址评论例子修改为多个生产者和消费者。除初始化第一轮外，线程的执行顺序变得随机，最终消费者取到的水果数也不相同。
```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * 通过阻塞队列实现生产者和消费者模式
 * 生产者向盘子(队列)放入苹果，消费者从盘子(队列)取走苹果
 * 盘子满则生产者阻塞等待，盘子空则消费者阻塞等待
 * 参考评论答案简单实现
 */
public class ProducerConsumerDemo {
    public static void main(String[] args) {
        Plate p = new Plate();
        for (int i = 0; i < 10; i++) {
            new Thread(new Producer(p,""+i)).start();
            new Thread(new Consumer(p,""+i)).start();
        }


    }
    
    static class Plate {
        // 一个盘子，可以放10个水果
        private BlockingQueue<String> fruits = new LinkedBlockingQueue(10);

        // 如果有水果，则取得，否则取不走
        public String get() {
            try {
                return fruits.take();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        }

        public void put(String fruit) {
            try {
                fruits.put(fruit);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    static class Producer implements Runnable {

        private Plate plate;
        private String name;
        public Producer(Plate p,String name) {
            this.plate = p;
            this.name = name;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    this.plate.put("" + i);
                    System.out.println("生产者"+ name +"第" + i + "个水果放入盘子");
                    Thread.sleep((long) (200 * Math.random()));
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }

    static class Consumer implements Runnable {
        private String name;
        private Plate plate;
        public Consumer(Plate p,String name) {
            this.plate = p;
            this.name = name;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    String j = this.plate.get();
                    System.out.println("消费者"+ name + "第" + j + "个水果取出盘子");
                    Thread.sleep((long) (200 * Math.random()));
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }
}

```
## 4. 多读少写的场景应该使用哪个并发容器，为什么使用它？比如你做了一个搜索引擎，搜索引擎每次搜索前需要判断搜索关键词是否在黑名单里，黑名单每天更新一次。
可以使用CopyOnWriteArrayList容器。因为在底层实现上采用数组，对读操作不加锁；针对写操作先复制一份当前容器数据，
然后对拷贝进行修改，最后通过CAS操作把对象引用指向新的数组。当然需要考虑数组拷贝的性能消耗。

# Java中的锁

## 1. 如何实现乐观锁（CAS）？如何避免ABA问题？
JVM中CAS操作是由底层CPU指令支持实现，在实现乐观锁时主要有三个步骤：
1.读取内存值，；2.比较内存值和期望值；3.如果相同则修改内存值，如果不同则循环重试。

ABA问题即数据A在被修改为B后又修改回了A，看上去像没有修改一样。
避免ABA问题可以采用版本号控制：每次修改都对版本号做+1操作，通过版本号比较确认数据是否改变过。
还可以采用Java并发包中AtomicStampedReference实现：底层为一个对象引用和整数对，可在修改时更新。

## 1. 读写锁可以用于什么应用场景？
读写锁适合于对数据读多写少的场景。Java中读写锁的实现ReentrantReadWriteLock：同一个锁不同视角下的使用。

## 1. 什么时候应该使用可重入锁？
在内部锁无法满足需求，需要使用一些更高级的特性时使用重入锁，重入锁相比内部锁可以实现限时等待tryLock、
条件等待lock.newCondition().await()，可中断锁lockInterruptibly，公平锁等
## 1. 什么场景下可以使用volatile替换synchronized？
仅需要保证可见性，没有原子性需求的场景；对volatile变量值的修改不依赖于原值。
# 并发工具

## 1. 如何实现一个流控程序，用于控制请求的调用次数？
可以考虑使用信号量Semaphore实现，通过信号量的个数控制请求的次数，在处理请求时首先尝试从获取一个信号量，
如果成功则处理，否则返回定制的错误信息。
如果是针对特定对象的请求控制，还可以考虑使用ConcurrentHashMap+LongAdder来存储请求计数。
