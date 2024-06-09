+++
title = 'Turbocharge Java Microservices with Quarkus and GraalVM Native Image'
date = 2024-01-07T11:04:18+03:00

images = ['images/Mandrel_specialized_distribution_GraalVM-01.png']

tags = ['quarkus', 'graalvm', 'microservice']

categories = ['Quarkus']

noComment = false
+++

In the dynamic landscape of modern software development, microservices have become the favored
architectural approach. While this methodology offers numerous advantages, it is not without its
challenges. Issues such as large memory footprints, extended start times, and high CPU usage often
accompany traditional JVM-based services. These challenges not only impact the technical aspects but
also have financial implications that can significantly affect the overall cost of running and
maintaining software solutions.

## What is GraalVM Native Image?

GraalVM Native Image is a key feature of the GraalVM, which is a high-performance runtime
that
provides support for various programming languages and execution modes. Specifically, GraalVM
Native Image allows you to compile Java applications ahead-of-time into standalone native
executables,
bypassing the need for a Java Virtual Machine (JVM) during runtime. This innovative approach
yields executable files that exhibit nearly instantaneous startup
times and significantly reduced memory consumption compared to their traditional JVM counterparts.
These native executables are meticulously crafted, containing only the essential classes, methods,
and dependent libraries indispensable for the application's functionality.
Beyond its technical prowess, GraalVM Native Image emerges as a strategic solution with
far-reaching
implications. It not only surmounts technical challenges but also introduces a compelling financial
case. By facilitating the development of efficient, secure, and instantly scalable cloud-native Java
applications, GraalVM becomes instrumental in optimizing resource utilization and fostering
cost-effectiveness. In essence, it plays a pivotal role in elevating the performance and financial
efficiency of software solutions in contemporary, dynamic environments.

## Technical Challenges and Financial Implications

**1. Large Memory Footprints**

**Technical Impact:** Traditional JVM-based services often incur substantial memory overhead due to
classloading and metadata for loaded classes.

**Financial Case:** High memory consumption translates to increased infrastructure costs. GraalVM's
elimination of metadata for loaded classes and other optimizations leads to a more efficient use of
resources, resulting in potential cost savings.

**2. Extended Start Times**

**Technical Impact:** Cold starts in microservices can lead to higher response times, impacting user
experience and potentially causing service degradation.

**Financial Case:** Extended start times not only affect user satisfaction but also contribute to
higher
operational costs. GraalVM's optimizations, such as eliminating classloading overhead and
pre-generating image heap during the build, drastically reduce startup times, potentially minimizing
operational expenses.

**3. High CPU Usage**

**Technical Impact:** Traditional JVMs often burn CPU cycles for profiling and Just-In-Time (JIT)
compilation during startup.

**Financial Case:** Excessive CPU usage results in increased cloud infrastructure costs. GraalVM's
avoidance of profiling and JIT-ing overhead directly contributes to reduced CPU consumption,
translating to potential cost savings in cloud usage.

### Tackling the Cold Start Problem

Microservices, especially in serverless or containerized environments, often face the Cold Start
Problem, impacting response times and user experience. GraalVM addresses this challenge by
implementing several optimizations:

**1. No Classloading Overhead**

- Traditional Java applications rely on classloading at runtime to dynamically load and link
  classes. This process introduces overhead, particularly during the startup phase. GraalVM
  minimizes this overhead through a process known as static or ahead-of-time (AOT) compilation. This
  involves pre-loading, linking, and partially initiating all classes that the application requires.
  As a result, there is no need for runtime classloading during application startup.

**2. Elimination of Interpreted Code**

- Traditional Java Virtual Machines rely on an interpreted execution mode before applying
  Just-In-Time (JIT) compilation. This can contribute to startup delays and increased CPU usage.
  Native executables contain no interpreted code, further contributing to faster startup times.

**3. No Profiling and JIT-ing Overhead**

- GraalVM bypasses the need to start the Just-In-Time (JIT) Compiler, reducing CPU usage
  during startup.

**4. Image Heap Generation at Build Time**

- GraalVM's native image utility enables the execution of initialization processes for
  specific classes during the build process. This results in the generation of an image heap that
  includes
  pre-initialized portions, speeding up the application's startup.

Oracle GraalVM's native image utility has demonstrated startup times almost 100 times faster
than traditional JVM-based applications. The graph below illustrates the substantial reduction in
runtime
memory requirements, showcasing GraalVM's efficiency compared to HotSpot(**Figure 1**).

<img src="/images/hello_word_graalvm.png" alt="Executables start up" title="Executables start up">
<br>

_Figure 1 – Native executables start up almost instantly(oracle.com)_

<br>

### Achieving a Leaner Memory Footprint

GraalVM contributes to lower memory footprints through the following optimizations:

**1. No Metadata for Loaded Classes**

- GraalVM avoids storing metadata for dynamically loaded classes in the non-heap memory. During
  the build process, the necessary class information is pre-loaded and linked, minimizing the need
  for additional metadata at runtime.

**2. No Profiling Data or JIT Optimizations**

- Since the bytecode is already in native code, GraalVM eliminates the need for collecting
  profiling data for JIT optimizations, reducing memory overhead.

**3. Isolation Technology**

- GraalVM introduces Isolates, a technology that partitions the heap into smaller, independent "
  heaps," enhancing efficiency, particularly in request processing scenarios.

In common, it consumes up to x5 times less memory compared to running on a JVM(**Figure 2**)

<img src="/images/memory_usage_graalvm.png" alt="Memory compared to Go or Java HotSpot" title="Memory compared to Go or Java HotSpot">
<br>

_Figure 2 – Native executables memory compared to Go or Java HotSpot(oracle.com)_

<br>

In conclusion, GraalVM's native image utility offers a transformative solution to the challenges
posed by microservices, addressing startup time, memory footprint, and CPU usage concerns. By
adopting GraalVM, developers can create cloud-native Java applications that are not only
efficient and secure but also provide a superior user experience.

## Native Java with Quarkus

To compile your Quarkus service into a native image, various methods are available. While this
article won't delve deeply into the Quarkus native build procedure, it does provide an overview of
the essential steps.

Before proceeding with any approach for building a native image, it's crucial to set up the proper
native profile in your `pom.xml` file. Add the following profile:

```xml {linenos=inline}
<profiles>
  <profile>
    <id>native</id>
    <properties>
      <quarkus.package.type>native</quarkus.package.type>
    </properties>
  </profile>
</profiles>
```

1. Producing a Native Executable with Installed GraalVM

Check your GraalVM version using the following command:

```shell {linenos=inline}
./gu info native-image
```

This command will display the installed GraalVM version:

```shell
Downloading: Component catalog from www.graalvm.org
Filename : https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-22.3.0/native-image-installable-svm-java19-linux-amd64-22.3.0.jar
Name     : Native Image
ID       : native-image
Version  : 22.3.0
GraalVM  : 22.3.0
Stability: Experimental
Component bundle native-image cannot be installed
        - The same component Native Image (org.graalvm.native-image[22.3.0.0/55b341ca1bca5219aafa8ed7c8a2273b81d184dd600d8261c837fc32d2dedae5]) is already installed in version 22.3.0
```

And to create a native executable, use:

```shell {linenos=inline}
./mvnw install -Dnative
```

These commands generate a `*-runner` binary in the `target` directory, allowing you to run the
native executable:

```shell {linenos=inline}
./target/*-runner
```

2. Creating a Native Executable without installed GraalVM

If installing GraalVM locally poses challenges, an in-container build can be used:

```shell {linenos=inline}
./mvnw install -Dnative -Dquarkus.native.container-build=true -Dquarkus.native.builder-image=graalvm
```

This command initiates the build within a Docker container and provides the necessary image file.
You can then start the application with:

```shell {linenos=inline}
./target/*-runner
```

In cases where building the native image proves challenging, the RedHat team provides a specialized
distribution of GraalVM designed for the Quarkus framework called Mandrel. Mandrel streamlines
GraalVM, focusing solely on the native-image capabilities essential for Quarkus applications. To
use Mandrel, follow these steps:

1. Identify the appropriate Mandrel version [Mandrel repository](https://quay.io/repository/quarkus/ubi-quarkus-mandrel-builder-image?tab=tags)

2. Set the Mandrel version in your `application.properties` file:

```properties {linenos=inline}
quarkus.native.builder-image=quay.io/quarkus/ubi-quarkus-mandrel-builder-image:23.0.1.2-Final-java17
```

3. Run the Maven build command:

```shell {linenos=inline}
./mvnw clean install -Pnative
```

### Manually Creating a Container

For those who prefer manual control over container creation, a multi-stage Docker build can be
employed.

```dockerfile {linenos=inline}
FROM quay.io/quarkus/ubi-quarkus-mandrel-builder-image:23.0.1.2-Final-java17 AS build
COPY --chown=quarkus:quarkus mvnw /app/mvnw
COPY --chown=quarkus:quarkus .mvn /app/.mvn
COPY --chown=quarkus:quarkus pom.xml /app/
USER quarkus
WORKDIR /app
RUN ./mvnw -B org.apache.maven.plugins:maven-dependency-plugin:3.6.1:go-offline
COPY src /app/src
RUN ./mvnw package -Dnative

FROM quay.io/quarkus/quarkus-micro-image:2.0
WORKDIR /app/
COPY --from=build /app/target/*-runner /app/application

RUN chmod 775 /app /app/application \
  && chown -R 1001 /app \
  && chmod -R "g+rwX" /app \
  && chown -R 1001:root /app

EXPOSE 8080
USER 1001

CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

This Dockerfile orchestrates a multi-stage build, resulting in a Docker image with your Quarkus
application. Execute this Dockerfile to produce the Docker image, ready to run your Quarkus
application.

## Summary

GraalVM Native Image is a powerful technology that can revolutionize the way you develop and deploy Java microservices. By adopting GraalVM Native Image, you can create microservices that are:

- Faster
- More scalable
- Simpler to deploy
- More cost-effective

GraalVM Native Image is a key enabler of cloud-native Java development and can help you achieve the performance, scalability, and cost savings that your business demands.

I hope this updated text is more helpful!