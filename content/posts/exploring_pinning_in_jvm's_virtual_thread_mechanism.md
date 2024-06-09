+++
title = "Exploring Pinning in JVM's Virtual Thread Mechanism"
date = 2024-03-28

images = ['images/vthreads.png']

tags = ['jvm', 'virtual threads', 'java']

categories = ['JVM']

noComment = false
+++

Java's virtual threads offer a lightweight alternative to traditional OS threads, enabling efficient
concurrency management. But understanding their behavior is crucial for optimal performance.
This blog post dives into pinning, a scenario that can impact virtual thread execution, and explores
techniques to monitor and address it.

## Virtual Threads: A Lightweight Concurrency Approach

Java's virtual threads are managed entities that run on top of the underlying operating system
threads (carrier threads). They provide a more efficient way to handle concurrency compared to
creating numerous OS threads, as they incur lower overhead. The JVM maps virtual threads to carrier
threads dynamically, allowing for better resource utilization.

* Managed by the JVM: Unlike OS threads that are directly managed by the operating system, virtual
  threads are created and scheduled by the Java Virtual Machine (JVM). This allows for finer-grained
  control and optimization within the JVM environment.

* Reduced Overhead: Creating and managing virtual threads incurs significantly lower overhead
  compared to OS threads. This is because the JVM can manage a larger pool of virtual threads
  efficiently, utilizing a smaller number of underlying OS threads.

* Compatibility with Existing Code: Virtual threads are designed to be seamlessly integrated with
  existing Java code. They can be used alongside traditional OS threads and work within the familiar
  constructs like Executor and ExecutorService for managing concurrent.

The figure below shows the relationship between virtual threads and platform threads:

<br>

<img src="/images/virtual_threads.jpg" alt="Virtual threads" title="Virtual threads">

<br>

## Pinning: When a Virtual Thread Gets Stuck

Pinning occurs when a virtual thread becomes tied to its carrier thread. This essentially means the
virtual thread cannot be preempted (switched to another carrier thread) while it's in a pinned
state. Here are common scenarios that trigger pinning:

* Synchronized Blocks and Methods: Executing code within a synchronized block or method leads to
  pinning. This ensures exclusive access to shared resources, preventing data corruption issues.

**Code Example:**

```java {linenos=inline}
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Main {

  public static void main(String[] args) throws InterruptedException {

    final Counter counter = new Counter();

    Runnable task = () -> {
      for (int i = 0; i < 1000; i++) {
        counter.increment();
      }
    };

    Thread thread1 = new Thread(task);
    Thread thread2 = new Thread(task);

    thread1.start();
    thread2.start();

    thread1.join();
    thread2.join();

    System.out.println("Final counter value: " + counter.getCount());
  }
}

class Counter {

  private int count = 0;

  public synchronized void increment() {
    count++;
  }

  public synchronized int getCount() {
    return count;
  }
}
```

In this example, when a virtual thread enters the `synchronized` block, it becomes pinned to its
carrier thread, but this is not always true.
Java's `synchronized` keyword alone is not enough to cause thread pinning in virtual
threads. For thread pinning to occur, there must be a blocking point within a `synchronized` block
that causes a virtual thread to trigger park, and ultimately disallows unmounting from its carrier
thread. Thread pinning could cause a decrease in performance as it would negate the benefits of
using lightweight/virtual threads.

Whenever a virtual thread encounters a blocking point, its state is transitioned to `PARKING`. 
This state transition is indicated by invoking the `VirtualThread.park()` method:

```java {linenos=inline}
// JDK core code
void park() {
  assert Thread.currentThread() == this;
  // complete immediately if parking permit available or interrupted
  if (getAndSetParkPermit(false) || interrupted)
    return;
  // park the thread
  setState(PARKING);
  try {
    if (!yieldContinuation()) {
      // park on the carrier thread when pinned
      parkOnCarrierThread(false, 0);
    }
  } finally {
    assert (Thread.currentThread() == this) && (state() == RUNNING);
  }
}
```

Let's take a look at a code sample to illustrate this concept:

```java {linenos=inline}
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Main {

  public static void main(String[] args) {

    Counter counter = new Counter();

    Runnable task = () -> {
      for (int i = 0; i < 100; i++) {
        counter.increment();
      }
    };

    Thread thread1 = Thread.ofVirtual().start(task);
    Thread thread2 = Thread.ofVirtual().start(task);

    try {
      thread1.join();
      thread2.join();
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    }

    System.out.println("Final counter value: " + counter.getCount());
  }
}

class Counter {

  private int count = 0;

  public void increment() {
    synchronized (this) {
      try {
        Thread.sleep(100); // This simulates a blocking call within the synchronized block
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      count++;
    }
  }

  public synchronized int getCount() {
    return count;
  }
}

```

* Native Methods/Foreign Functions: Running native methods or foreign functions can also cause
  pinning. The JVM might not be able to efficiently manage the virtual thread's state during these
  operations.

<br> 

## Monitoring Pinning with `-Djdk.tracePinnedThreads=full`

The `-Djdk.tracePinnedThreads=full` flag is a JVM startup argument that provides detailed tracing
information about virtual thread pinning. When enabled, it logs events like:

- Virtual thread ID involved in pinning
- Carrier thread ID to which the virtual thread is pinned
- Stack trace indicating the code section causing pinning

{{< notice note >}}

Use this flag judiciously during debugging sessions only, as it introduces performance overhead.

{{< /notice >}}

1. Compile the our demo code:

```shell {linenos=inline}
javac Main.java
```

2. Star the compiled code with the `-Djdk.tracePinnedThreads=full` flag:

```shell {linenos=inline}
java -Djdk.tracePinnedThreads=full Main
```

3. Observe the output in the console, which shows detailed information about virtual thread
   pinning.:

```shell {linenos=inline}
Thread[#29,ForkJoinPool-1-worker-1,5,CarrierThreads]
java.base/java.lang.VirtualThread$VThreadContinuation.onPinned(VirtualThread.java:183)
java.base/jdk.internal.vm.Continuation.onPinned0(Continuation.java:393)
java.base/java.lang.VirtualThread.parkNanos(VirtualThread.java:621)
java.base/java.lang.VirtualThread.sleepNanos(VirtualThread.java:791)
java.base/java.lang.Thread.sleep(Thread.java:507)
Counter.increment(Main.java:38) <== monitors:1
Main.lambda$main$0(Main.java:13)
java.base/java.lang.VirtualThread.run(VirtualThread.java:309)

Final counter value: 200

```

<br> 

## Fixing Pinning with Reentrant Locks

Pinning is an undesirable scenario which impedes the performance of virtual threads. Reentrant locks serve as an effective tool to counteract pinning.
Here's how you can use `Reentrant` locks to mitigate pinning situations:

```java {linenos=inline}
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

public class Main {

  public static void main(String[] args) {

    Counter counter = new Counter();

    Runnable task = () -> {
      for (int i = 0; i < 100; i++) {
        counter.increment();
      }
    };

    Thread thread1 = Thread.ofVirtual().start(task);
    Thread thread2 = Thread.ofVirtual().start(task);

    try {
      thread1.join();
      thread2.join();
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    }

    System.out.println("Final counter value: " + counter.getCount());
  }
}

class Counter {

  private int count = 0;
  private final ReentrantLock lock = new ReentrantLock();

  public void increment() {
    lock.lock();
    try {
      Thread.sleep(100); // This simulates a blocking call
      count++;
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      lock.unlock();
    }
  }

  public int getCount() {
    return count;
  }
}
```

In the updated example, we use a `ReentrantLock` instead of a `synchronized` block. The thread can
acquire the lock and release it immediately after it completes its operation,
potentially reducing the duration of pinning compared to a `synchronized` block which might hold the
lock for a longer period.

## In Conclusion

Java's virtual threads stand as a testimony to the evolution and the capabilities of the language.
They offer a fresh, lightweight alternative to traditional OS threads, providing a bridge to
efficient concurrency management. Taking the time to dig deep and understand key concepts such as
thread pinning can equip developers with the know-how to leverage the full potential of these
lightweight threads. This knowledge not only prepares developers for leveraging upcoming features
but also empowers them to resolve complex concurrency control issues more effectively in their
current projects.