+++
title = 'Memory Management in GraalVM Native Image'
date = 2024-05-26T11:04:18+03:00

images = ['images/graalvm-common.png']

tags = ['native', 'graalvm', 'compiler', 'memory management']

categories = ['GraalVM']

noComment = false
+++

<style>
.zoom {
  transition: transform .2s; /* Animation */
  margin: 0 auto;
}

.zoom:hover {
  transform: scale(2.0); /* Zoom when hovered */
}
</style>

Memory management is a crucial component of computer software development, tasked with the effective
allocation, utilization, and release of memory in applications. Its importance lies in enhancing
software performance and ensuring system stability.

## Garbage Collection

Garbage collection (GC) is pivotal in contemporary programming languages such as Java and Go. It
autonomously detects and recycles unused memory, thereby alleviating the need for developers to
manually manage memory. The concept of GC originally emerged in the LISP programming language in the
late 1950s, marking the introduction of automated memory management.

Key advantages of automated memory management include:

- Prevention of memory leaks and efficient memory utilization.
- Simplified development processes and enhanced program stability.

Understanding the nature of "garbage" in memory and identifying reclaimable space is essential. In the upcoming chapters, we will start by exploring the fundamental principles of garbage collection.

### Reference Counting Algorithm [George E. Collins 1966]

The Reference Counting Algorithm assigns a field in the object's header to track its reference
count. This count increases with each new reference and decreases when a reference is removed. When
the count reaches zero, the object is eligible for garbage collection.

Consider the following code:

First create a `String` with value `demo` which is referenced by `d`  (**Figure 1**).
```java {linenos=inline}
String d = new String("demo");
```

<br>

<img class="zoom" src="/images/gc_01.png" alt="Reference Counting" title="Reference Counting">

_Figure 1 – After a String is created_

Then, set `d` to `null`. The reference count of `demo` is zero. In the Reference Counting algorithm,
the memory for `demo` is to be reclaimed (**Figure 2**).

```java
d =null; // Reference count of 'demo' becomes zero, prompting garbage collection.
```
<br>

<img class="zoom" src="/images/gc_02.png" alt="Reference Counting" title="Reference Counting">

_Figure 2 – When the reference is nullified_

The Reference Counting Algorithm operates during program execution, avoiding __Stop-The-World__
events, which halt the program temporarily for garbage collection. However, its major drawback is
the inability to handle circular references (**Figure 3**).

For example:

```java {linenos=inline}
public class CircularReferenceDemo {

  public CircularReferenceDemo reference;
  private String name;

  public CircularReferenceDemo(String name) {
    this.name = name;
  }

  public void setReference(CircularReferenceDemo ref) {
    this.reference = ref;
  }

  public static void main(String[] args) {
    CircularReferenceDemo objA = new CircularReferenceDemo("Ref_A");
    CircularReferenceDemo objB = new CircularReferenceDemo("Ref_B");

    objA.setReference(objB);
    objB.setReference(objA);

    objA = null;
    objB = null;
  }
}
```

Here, despite nullifying external references, the mutual references between `objA` and `objB`
prevent their garbage collection.

<br>

<img class="zoom" src="/images/gc_03.png" alt="Reference Counting" title="Reference Counting">

_Figure 3 – Circular References_

We can see that both objects can no longer be accessed. However, they are referenced by each other,
and thus their reference count will never be zero. Consequently, the GC collector will never be
notified to garbage collect them by using the Reference Counting algorithm.

This algorithm is practically implemented in C++ through the use of `std::shared_ptr`. Designed to
manage the lifecycle of dynamically allocated objects, `std::shared_ptr` automates the increment and
decrement of reference counts as pointers to the object are created or destroyed.
This smart pointer is part of the C++ Standard Library, providing robust memory management
capabilities that significantly diminish the risks associated with manual memory handling.
Whenever a `std::shared_ptr` is copied, the internal reference count of the managed object
increases, reflecting the new reference. Conversely, when a `std::shared_ptr` is destructed, goes out
of scope, or is reassigned to a different object, the reference count decreases. The allocated
memory is automatically reclaimed and the object is destroyed when its reference count reaches zero,
effectively preventing memory leaks by ensuring no object remains allocated without necessity.

### Reachability Analysis Algorithm [1978]

The Reachability Analysis Algorithm begins at GC roots, traversing through the object graph. Objects
that cannot be reached from these roots are deemed unrecoverable and are targeted for collection.

As shown in the image below, the objects in blue circle should be kept alive and the objects in
gray circle can be recycled (**Figure 4**).

<br>
<img class="zoom" src="/images/gc_04.png" alt="Reachability Analysis" title="Reachability Analysis">

_Figure 4 – Memory leak_

This method effectively resolves the issue of circular references inherent in the Reference Counting
Algorithm. Objects unreachable from the GC roots are categorized for collection.

Typically, Java objects considered as GC roots include:

- Local variables within the current method scope.
- Active Java threads.
- Static fields from classes.
- JNI references from native code.

## Overview of GraalVM Native Image

GraalVM offers an ahead-of-time (AOT) compiler, which translates Java applications into standalone
executable binaries known as GraalVM Native Images. Developed by Oracle Labs, these binaries
encapsulate application and library classes, and runtime components like the GC, allowing operations
without a Java Runtime Environment (JRE).

The process involves static analysis to determine reachable components, initialization through
executed blocks, and finalizing by creating a snapshot of the application state for subsequent
machine code translation.

## Fundamentals of the Substrate VM

The Substrate VM stands as an integral part of the GraalVM suite, orchestrated by Oracle Labs. It's
an enhanced JVM that not only supports ahead-of-time (AOT) compilation but also facilitates the
execution of languages beyond Java, such as JavaScript, Python, Ruby, and even native languages like
C and C++. At its core, Substrate VM serves as a sophisticated framework that allows GraalVM to
compile Java applications into standalone native binaries. These binaries do not rely on a
conventional Java Virtual Machine (JVM) for their execution, which streamlines deployment and
operational processes.

One of the cardinal features of Substrate VM is its specialized garbage collector, which is
fine-tuned for applications requiring low latency and minimal memory footprint. This garbage
collector is adept at handling the unique memory layout and operational model distinct to native
images, which differ considerably from traditional Java applications running on a standard JVM. The
absence of a Just-In-Time (JIT) compiler in Substrate VM native images is a strategic choice that
aids in minimizing the overall size of the executable. This is because it eliminates the necessity
to include the JIT compiler and associated metadata, which are substantial in size and complexity.

Furthermore, while GraalVM is developed using Java, this introduces certain constraints,
particularly in terms of native memory access. Such restrictions are primarily due to security
concerns and the need to maintain compatibility across various platforms. However, accessing native
memory is essential for optimal garbage collection operations. To address this, Substrate VM employs
a suite of specialized interfaces that facilitate safe and efficient interactions with native
memory. These interfaces are part of the broader GraalVM architecture and enable Substrate VM to
effectively manage memory in a manner akin to lower-level languages like C, all while retaining the
safety and manageability of Java.

In practice, these capabilities make Substrate VM an extremely versatile tool that enhances the
functionality and efficiency of applications compiled with GraalVM. By allowing developers to
leverage a broader range of programming languages and compile them into efficient native binaries,
Substrate VM pushes the boundaries of what can be achieved with traditional Java development
environments. This makes it an invaluable asset for modern software development projects that demand
high performance, reduced resource consumption, and versatile language support.

Noteworthy elements of Substrate VM include:

- Simplified memory access via interfaces like `Pointer` [Interface Pointer](https://www.graalvm.org/sdk/javadoc/org/graalvm/word/Pointer.html) for raw memory operations and `WordBase`
  [Interface WordBase](https://www.graalvm.org/sdk/javadoc/org/graalvm/word/WordBase.html) 
  for handling word-sized values.

- Division of the heap into pre-initialized segments containing immutable objects and runtime
  segments for dynamic object allocation (**Figure 5**).

<br>
<img class="zoom" src="/images/gc_05.png" alt="Memory Management in Native Image" title="Memory Management in Native Image">

_Figure 5 – Memory Management in Native Image_

At runtime, the so-called image heap in Substrate VM contains objects created during the image build
process. This section of the heap is pre-initialized with data from the executable binary's data
section and is readily accessible upon application startup. The objects residing in the image heap
are considered immortal; hence, references within these objects are treated as root pointers by the
garbage collector. However, the GC only scans parts of the image heap for root pointers,
specifically those that are not marked as read-only.

During the build process, objects designated as read-only are placed in a specific read-only section
of the image heap. Since these objects will never hold references to objects allocated at runtime,
they contain no root pointers, allowing the GC to bypass them during scans. Similarly, objects that
solely consist of primitive data or arrays of primitive types also lack root pointers. This
attribute further streamlines the garbage collection process, as these objects can be omitted from
GC scans.

In contrast, the Java heap is designated for holding ordinary objects that are created dynamically
during runtime. This portion of the heap is subject to regular garbage collection to reclaim space
occupied by objects that are no longer in use. It is structured as a generational heap with
mechanisms for aging, facilitating efficient memory management over time.

This division between the pre-initialized, immortal image heap and the dynamically managed Java heap
enables Substrate VM to optimize memory usage and garbage collection efficiency, catering to both
static and dynamic aspects of application memory requirements.

## Heap Chunk

In Substrate VM's heap model, the memory is systematically organized into structures known as heap
chunks. These chunks, typically sized at 1024KB by default, form a continuous segment of virtual
memory that is solely allocated to object storage. The organizational structure of these chunks is a
linked list where the tail chunk represents the most recently added segment. Such a model
facilitates efficient memory allocation and object management.

These heap chunks are further categorized into two types: aligned and unaligned. Aligned heap chunks
are capable of holding multiple objects continuously. This alignment allows for simpler mapping of
objects to their respective parent heap chunks, making memory management more intuitive and
efficient. In scenarios where object promotion is necessary-typically, during garbage collection and
memory optimization- an object is moved from its original placement in a parent heap chunk to a
target heap chunk located in a designated "old to-space." This migration is part of the generational
heap management strategy that helps in optimizing the garbage collection process by segregating
young from old objects, thereby reducing the overhead during GC cycles.

## Garbage Collectors in Native Image

GraalVM Native Image supports various GCs tailored to different needs:

- __Serial GC:__ Default low-footprint collector suitable for single-threaded applications.

- __G1 Garbage Collector:__ Designed for multi-threaded applications with large heap sizes,
  enhancing flexibility in generation management.

- __Epsilon GC:__ A minimalistic collector that handles allocation but lacks reclamation, best used
  for short-lived applications where full heap utilization is predictable.

## Conclusion

In conclusion, Substrate VM effectively optimizes memory management within GraalVM by incorporating
advanced techniques like specialized garbage collection and structured heap management. These
features, including heap chunks and separate memory segments for image and Java heaps, streamline
garbage collection and improve application performance. As Substrate VM supports a variety of
programming languages and compiles them into efficient native binaries, it showcases how modern JVM
frameworks can extend beyond traditional boundaries to enhance execution efficiency and robustness
in diverse application environments. This approach sets a high standard for future developments in
virtual machine technology and application deployment.