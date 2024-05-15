+++
title = 'Enhancing Performance with Static Analysis, Image Initialization and Heap Snapshotting'
date = 2024-05-15T11:04:18+03:00

images = ['images/native-image-header-bf334c66ff01cbe5afcbe5325ea9eb79ef21cb258213f9d95fe0d3d2b0d9bd0a.png']

tags = ['native', 'graalvm', 'compiler']

categories = ['GraalVM']

noComment = false
+++

From monolithic structures to the world of distributed systems, application development has come a
long way. The massive adoption of cloud computing and microservice architecture has significantly
altered the approach to how server applications are created and deployed. Instead of giant
application servers, we now have independent, individually deployed services that spring into action
as and when needed.

However, a new player on the block that can impact this smooth functioning might be __'cold starts.'__
Cold starts kick in when the first request processes on a freshly spawned worker. This situation
demands language runtime initialization and service configuration initialization before processing
the actual request. The unpredictability and slower execution associated with cold starts can breach
the service level agreements of a cloud service. So, how does one counter this growing concern?

<style>
.zoom {
  transition: transform .2s; /* Animation */
  margin: 0 auto;
}

.zoom:hover {
  transform: scale(2.0); /* Zoom when hovered */
}
</style>

## Native Image: Optimizing Startup Time and Memory Footprint

To combat the inefficiencies of cold starts, a novel approach has been developed involving points-to
analysis, application initialization at build time, heap snapshotting, and ahead-of-time  __(AOT)__
compilation. This method operates under a closed-world assumption, requiring all Java classes to be
predetermined and accessible at build time. During this phase, a comprehensive points-to analysis
determines all reachable program elements (classes, methods, fields) to ensure that only essential
Java methods are compiled.

The initialization code for the application can execute during the build process rather than at
runtime. This allows for the pre-allocation of Java objects and the construction of complex data
structures, which are then made available at runtime via an  "image heap". This image heap is
integrated within the executable, providing immediate availability upon application start. The
iterative execution of points-to analysis and snapshotting continues until a stable state (fixed
point) is achieved, optimizing both startup time and resource consumption.

## Detailed Workflow

The input for our system is Java bytecode, which could originate from languages like Java, Scala, or
Kotlin. The process treats the application, its libraries, the JDK, and VM components uniformly to
produce a native executable specific to an operating system and architecture—termed a "native
image". The building process includes iterative points-to analysis and heap snapshotting until a
fixed point is reached, allowing the application to actively participate through registered
callbacks. These steps are collectively known as the native image build process (**Figure 1**)

<img class="zoom" src="/images/native-image-build-process.jpg" alt="Native Image Build Process" title="Native Image Build Process">

<br>

_Figure 1 – Native Image Build Process(source: redhat.com)_

## Points-to Analysis

We employ a points-to analysis to ascertain the reachability of classes, methods, and fields during
runtime. The points-to analysis commences with all entry points, such as the main method of the
application, and iteratively traverses all transitively reachable methods until reaching a fixed
point(**Figure 2**).

<img class="zoom" src="/images/points-to-analysis.jpg" alt="Points-to-analysis" title="Points-to-analysis">

<br>

_Figure 2 – Points-to-analysis_

Our points-to analysis leverages the front end of our compiler to parse Java bytecode into the
compiler’s high-level intermediate representation __(IR)__. Subsequently, the IR is transformed into
a
type-flow graph. In this graph, nodes represent instructions operating on object types, while edges
denote directed use edges between nodes, pointing from the definition to the usage. Each node
maintains a type state, consisting of a list of types that can reach the node and nullness
information. Type states propagate through the use edges; if the type state of a node changes, this
change is disseminated to all usages. Importantly, type states can only expand; new types may be
added to a type state, but existing types are never removed. This mechanism ensures that the
analysis ultimately converges to a fixed point, leading to termination.

## Run Initialization Code

The points-to analysis guides the execution of initialization code when it hits a local fixed point.
This code finds its origins in two separate sources: Class initializers and custom code batch
executed at build time through a feature interface:

1. __Class Initializers:__ Every Java class can have a class initializer indicated by a `<clinit>`
   method,
   which initializes static fields. Developers can choose which classes to initialize at build-time
   vs runtime.

2. __Explicit Callbacks:__ Developers can implement custom code through hooks provided by our
   system,
   executing before, during, or after the analysis stages.

Here are the APIs provided for integrating with our system:

### Passive API (queries the current analysis status)

```java {linenos=inline}
boolean isReachable(Class<?> clazz);

boolean isReachable(Field field);

boolean isReachable(Executable method);
```

For more information, refer to
the [QueryReachabilityAccess](https://github.com/graalvm/labs-openjdk/blob/378138863fe29bae72f34eb8e3af8ab7c457baa6/src/jdk.internal.vm.ci/share/classes/jdk/vm/ci/runtime/JVMCICompiler.java#L35)

### Active API (registers callbacks for analysis status changes):

```java {linenos=inline}
void registerReachabilityHandler(Consumer<DuringAnalysisAccess> callback, Object... elements);

void registerSubtypeReachabilityHandler(BiConsumer<DuringAnalysisAccess, Class<?>> callback, Class<?> baseClass);

void registerMethodOverrideReachabilityHandler(BiConsumer<DuringAnalysisAccess, Executable> callback, Executable baseMethod);
```

For more information, refer to
the [BeforeAnalysisAccess](https://github.com/oracle/graal/blob/979124badd31e91224996ddd08aaf2e10bfeb37d/sdk/src/org.graalvm.nativeimage/src/org/graalvm/nativeimage/hosted/Feature.java#L202)

During this phase, the application can execute custom code such as object allocation and
initialization of larger data structures. Importantly, the initialization code can access the
current points-to analysis state, enabling queries regarding the reachability of types, methods, or
fields. This is accomplished using the various `isReachable()` methods provided by
DuringAnalysisAccess. Leveraging this information, the application can construct data structures
optimized for the reachable segments of the application.

## Heap Snapshotting

Finally, heap snapshotting constructs an object graph by following root pointers like static fields
to build a comprehensive view of all reachable objects. This graph then populates the native image's
image heap, ensuring that the application's initial state is efficiently loaded upon startup.

To generate the transitive closure of reachable objects, the algorithm traverses object fields,
reading their values using reflection. It's crucial to note that the image builder operates within
the Java environment. Only instance fields marked as "read" by the points-to analysis are considered
during this traversal. For instance, if a class has two instance fields but one isn't marked as
read, the object reachable through the unmarked field is excluded from the image heap.

When encountering a field value whose class hasn't been previously identified by the points-to
analysis, the class is registered as a field type. This registration ensures that in subsequent
iterations of the points-to analysis, the new type is propagated to all field reads and transitive
usages in the type-flow graph.

The code snippet below outlines the core algorithm for heap snapshotting:

```text
Declare List worklist := []
Declare Set reachableObjects := []

Function BuildHeapSnapshot(PointsToState pointsToState)
For Each field in pointsToState.getReachableStaticObjectFields()
Call AddObjectToWorkList(field.readValue())
End For

    For Each method in pointsToState.getReachableMethods()
        For Each constant in method.embeddedConstants()
            Call AddObjectToWorkList(constant)
        End For
    End For

    While worklist.isNotEmpty
        Object current := Pop from worklist
        If current Object is an Array
            For Each value in current
                Call AddObjectToWorkList(value)
                Add current.getClass() to pointsToState.getObjectArrayTypes()
            End For
        Else
            For Each field in pointsToState.getReachableInstanceObjectFields(current.getClass())
                Object value := field.read(current)
                Call AddObjectToWorkList(value)
                Add value.getClass() to pointsToState.getFieldValueTypes(field)
            End For
        End If
    End While
    Return reachableObjects
End Function
```

In summary, the heap snapshotting algorithm efficiently constructs a snapshot of the heap by
systematically traversing reachable objects and their fields. This ensures that only relevant
objects are included in the image heap, optimizing the performance and memory footprint of the
native image.

## Conclusion

In conclusion, the process of heap snapshotting plays a critical role in the creation of native
images. By systematically traversing reachable objects and their fields, the heap snapshotting
algorithm constructs an object graph that represents the transitive closure of reachable objects
from root pointers such as static fields. This object graph is then embedded into the native image
as the image heap, serving as the initial heap upon native image startup.

Throughout the process, the algorithm relies on the state of the points-to analysis to determine
which objects and fields are relevant for inclusion in the image heap. Objects and fields marked
as "read" by the points-to analysis are considered, while unmarked entities are excluded.
Additionally, when encountering previously unseen types, the algorithm registers them for
propagation in subsequent iterations of the points-to analysis.

Overall, heap snapshotting optimizes the performance and memory usage of native images by ensuring
that only necessary objects are included in the image heap. This systematic approach enhances the
efficiency and reliability of native image execution.