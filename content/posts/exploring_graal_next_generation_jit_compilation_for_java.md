+++
title = 'Exploring Graal: Next-Generation JIT Compilation for Java'
date = 2024-05-06T11:04:18+03:00

images = ['images/graalvm-logo.png']

tags = ['JIT', 'graalvm', 'compiler']

categories = ['GraalVM']

noComment = false
+++


The Graal compiler is a radical leap forward in dynamic, Just-In-Time (JIT) compilation technology.
Heralded as a significant factor behind Java's impressive performance, the role and function of JIT
compilation within the Java Virtual Machine (JVM) architecture often perplexes many practitioners
due to its complex and rather opaque nature.

<style>
.zoom {
  transition: transform .2s; /* Animation */
  margin: 0 auto;
}

.zoom:hover {
  transform: scale(2.0); /* Zoom when hovered */
}
</style>
## What is a JIT compiler?

When you execute the javac command or use the IDE, your Java program is converted from Java source
code into JVM bytecode. This
process creates a binary representation of your Java program - a format much simpler and more
compact than the original source code.

Classical processors found in your computer or server, however, are incapable of executing JVM
bytecode directly. This necessitates the JVM to interpret the bytecode.

<img class="zoom" src="/images/jit-compiler.jpg" alt="How a just-in-time (JIT) compiler works" title="How a just-in-time (JIT) compiler works">

<br>

_Figure 1 – How a just-in-time (JIT) compiler works_

Interpreters can often underperform compared to native code running on an actual processor, which
motivates the JVM to invoke another compiler at runtime - the JIT compiler. The JIT compiler
translates your bytecode into machine code that your processor can run directly. This sophisticated
compiler executes a range of advanced optimizations to generate high-quality machine code.

This bytecode acts as an intermediate layer, enabling Java applications to run on various operating
systems with different processor architectures. The JVM itself is a software program that interprets
this bytecode instruction by instruction.

## The Graal JIT Compiler – It’s Written in Java

The OpenJDK implementation of the JVM contains two conventional JIT-compilers – the Client
Compiler (C1) and the Server Compiler (C2 or Opto). The Client Compiler is optimized for faster
operation and less optimized code output, making it ideal for desktop applications where extended
JIT-compilation pauses can interrupt user experience. Conversely, the Server Compiler is engineered
to spend more time producing highly optimized code, making it suitable for long-running server
applications.

The two compilers can be used in tandem through "tiered compilation". Initially, the code is
compiled through C1, followed by C2 if execution frequency justifies the additional compilation
time.

Developed in C++, C2, despite its high-performance characteristics, has inherent downsides. C++ is
an unsafe language; therefore, errors in the C2 module could cause the entire VM to crash. The
complexity and rigidity of the inherited C++ code have also resulted in its maintenance and
extendibility becoming a significant challenge.

Unique to Graal, this JIT-compiler is developed in Java. The compiler's main requirement is
accepting JVM bytecode and outputting machine code – a high-level operation that doesn’t require a
system-level language like C or C++.

Graal being written in Java offers several advantages:

- **Improved safety:** Java's garbage collection and managed memory approach eliminate the risk of
  memory-related crashes from the JIT compiler itself.

- **Easier maintenance and extension:** The Java codebase is more approachable for developers to
  contribute to and extend the capabilities of the JIT compiler.

- **Portability:** Java's platform independence translates to the Graal JIT compiler potentially
  working
  on any platform with a Java Virtual Machine.

## The JVM Compiler Interface(JVMCI)

The JVM Compiler Interface (JVMCI) is an innovative feature and a new interface in the JVM (JEP
243: https://openjdk.org/jeps/243).
Much like the Java annotation processing API, the JVMCI also permits the integration of a custom
Java JIT compiler.

The JVMCI interface comprises a pure function from byte to `byte[]` :

```java {linenos=inline}
interface JVMCICompiler {

  byte[] compileMethod(byte[] bytecode);
}
```

This doesn't capture the full complexity of real-life scenarios.

In practical applications, we frequently need additional information such as the number of local
variables, the stack size, and data gathered from profiling in the interpreter to better understand
how the code is performing. Hence, the interface takes a more complex input. Instead of just
bytecode, it accepts a `CompilationRequest`:

```java {linenos=inline}
public interface JVMCICompiler {

  int INVOCATION_ENTRY_BCI = -1;

  CompilationRequestResult compileMethod(CompilationRequest request);
}
```

[JVMCICompiler.java](https://github.com/graalvm/labs-openjdk/blob/378138863fe29bae72f34eb8e3af8ab7c457baa6/src/jdk.internal.vm.ci/share/classes/jdk/vm/ci/runtime/JVMCICompiler.java#L35)

A `CompilationRequest` encapsulates more comprehensive information, such as which JavaMethod is
intended for compilation, and potentially much more data needed by the compiler.

[CompilationRequest.java](https://github.com/graalvm/labs-openjdk/blob/master/src/jdk.internal.vm.ci/share/classes/jdk/vm/ci/code/CompilationRequest.java)

This approach has the benefit of providing all necessary details to the custom JIT-compiler in a
more organized and contextual manner. To create a new JIT-compiler for the JVM, one must implement
the `JVMCICompiler` interface.

## Ideal Graph

An aspect where Graal truly shines in terms of performing sophisticated code optimization is in its
use of a unique data structure: the program-dependence-graph, or colloquially, an "Ideal Graph".

The program-dependence-graph is a directed graph that presents a visual representation of the
dependencies between individual operations, essentially laying out the matrix of dependencies
between different parts of your Java code.

Let's illustrate this concept with a simple example of adding two local variables, `x` and `y`.
The program-dependence-graph for this operation in Graal's context would involve three nodes and two
edges:

- **Nodes:**
    - `Load(x)` and `Load(y)`: These nodes represent the operations of loading the values of
      variables `x` and `y` from memory into registers within the processor.
    - `Add`: This node embodies the operation of adding the values loaded from `x` and `y`.

- **Edges:**
    - Two edges would be drawn from the `Load(x)` and `Load(y)` nodes to the Add node. These
      directional paths convey the data flow. They signify that the values loaded from `x` and `y`
      are the inputs to the addition operation.

```text
      +--------->+--------->+
      | Load(x)  | Load(y)  |
      +--------->+--------->+
                 |
                 v
              +-----+
              | Add |
              +-----+
```

In this illustration, the arrows represent the data flow between the nodes. The `Load(x)` and `Load(y)`
nodes feed their loaded values into the Add node, which performs the addition operation. This visual
representation helps Graal identify potential optimizations based on the dependencies between these
operations.

This graph-based architecture provides the Graal compiler with a clear visible landscape of
dependencies and scheduling in the code it compiles. The program-dependence-graph not only maps the
flow of data and relationships between operations but also offers a canvas for Gaal to manipulate
these relationships. Each node on the graph is a clear candidate for specific optimizations, while
the edges indicate where alterations would propagate changes elsewhere in the code - both aspects
influence how Graal optimizes your program's performance.

Visualizing and analyzing this graph can be achieved through a tool called
the `IdealGraphVisualizer`,
or IGV. This tool is invaluable in understanding the intricacies of Graal's code optimization
capabilities. It allows you to pinpoint how specific parts of your code are being analyzed,
modified, and optimized, providing valuable insights for further code enhancements.

Let's consider a simple Java program that performs a complex operation in a loop:

```java {linenos=inline}
public class Demo {
 public static void main(String[] args) {
        for (int i = 0; i < 1_000_000; i++) {
            System.err.println(complexOperation(i, i + 2));
        }
    }

    public static int complexOperation(int a, int b) {
        return ((a + b)-a) / 2;
    }
}
```
When compiled with Graal, the Ideal Graph for this program would look something like this(**Figure 2**).

<img class="zoom" src="/images/graal-graph.png" alt="Graal Graphs" title="Graal Graphs">

<br>

_Figure 2 – Graal Graphs_

Therefore, along with its method level optimizations and overall code performance improvements, this
graph-based representation constitutes the key to understanding the power of the Graal compiler in
optimizing your Java applications

## In Conclusion

The Graal JIT compiler represents a significant leap forward in Java performance optimization. Its unique characteristic of being written in Java itself offers a compelling alternative to traditional C-based compilers. This not only enhances safety and maintainability but also paves the way for a more dynamic and adaptable JIT compilation landscape.

The introduction of the JVM Compiler Interface (JVMCI) further amplifies this potential. By allowing the development of custom JIT compilers in Java, JVMCI opens doors for further experimentation and innovation. This could lead to the creation of specialized compilers targeting specific needs or architectures, ultimately pushing the boundaries of Java performance optimization.

In essence, Graal and JVMCI represent a paradigm shift in JIT compilation within the Java ecosystem. They lay the foundation for a future where JIT compilation can be customized, extended, and continuously improved, leading to even more performant and versatile Java applications.



