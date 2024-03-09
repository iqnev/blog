+++
title = 'Why we discarded Reactive systems architecture from our code?'
date = 2024-03-10

images = ['images/reactive.png']

tags = ['design', 'reactive']

categories = ['design']

noComment=false
+++

This article explores our decision to move away from reactive architecture in our software project. We'll delve into the core principles of reactive systems, the benefits of non-blocking I/O, and the challenges we faced with a reactive approach.

## Understanding Reactive architecture style

Reactive encompasses a set of principles and guidelines aimed at constructing responsive distributed systems and applications, characterized by:

1. **Responsiveness:** Capable of swiftly handling requests, even under heavy loads.
2. **Resilience:** Able to recover from failures with minimal downtime.
3. **Elasticity:** Can adapt to changing workloads by scaling resources accordingly.
4. **Message-Driven:** Utilizes asynchronous messaging to enhance fault tolerance and decouple components.

One key benefit of reactive systems is their use of non-blocking I/O. This approach avoids blocking threads during I/O operations, allowing a single thread to handle multiple requests concurrently. This can significantly improve system efficiency compared to traditional blocking I/O.
In traditional multithreading, blocking operations pose significant challenges in optimizing systems (**Figure 1**).
Greedy applications consuming excessive memory are inefficient and penalize other applications, often necessitating requests for additional resources like memory, CPU, or larger virtual machines.

<br>

<img src="/images/traditional-multithreading.jpg" alt="Traditional Multi-threading" title="Traditional Multi-threading">

<br>

_Figure 1 – Traditional Multi-threading_

<br>

I/O operations are integral to modern systems, and efficiently managing them is paramount to prevent greedy behavior.
Reactive systems employ non-blocking I/O, enabling a low number of OS threads to handle numerous concurrent I/O operations.

<br>

## Reactive Execution Model

Although non-blocking I/O offers substantial benefits, it introduces a novel execution model distinct from traditional frameworks.
Reactive programming emerged to address this issue, as it mitigates the inefficiency of platform threads idling during blocking operations (**Figure 2**).

<br>

<img src="/images/reactive-event-loop.jpg" alt="Reactive Event Loop" title="Reactive Event Loop">

<br>

_Figure 2 – Reactive Event Loop_

<br>

## Quarkus and Reactive

Quarkus leverages a reactive engine powered by Eclipse Vert.x and Netty, facilitating non-blocking I/O interactions.
Mutiny, the preferred approach for writing reactive code with Quarkus, adopts an event-driven paradigm, wherein reactions are triggered by received events.

<a href="https://smallrye.io/smallrye-mutiny/latest/">Mutiny</a> offers two event-driven and lazy types:

1. **Uni:** Emits a single event (an item or a failure), suitable for representing asynchronous actions with zero or one result.
2. **Multi:** Emits multiple events (n items, one failure, or one completion), representing streams of items, potentially unbounded.

<br>

## Challenges with Reactive

While reactive systems offer benefits, we encountered several challenges during development:

1. **Paradigm Shift:** Reactive programming necessitates a fundamental shift in developers' mindsets, which can be challenging, especially for developers accustomed to imperative programming.
   Unlike auxiliary tools like the Streams API, the reactive approach demands a complete mindset overhaul.
2. **Code Readability and Understanding:** Reactive code poses difficulties for new developers to comprehend, leading to increased time spent deciphering and understanding it. The complexity introduced by reactive paradigms compounds this issue.

<br>

{{< notice note >}}
"Indeed, the ratio of time spent reading versus writing is well over 10 to 1. We are constantly reading old code as part of the effort to write new code. ...[Therefore,] making it easy to read makes it easier to write."
**― Robert C. Martin, Clean Code: A Handbook of Agile Software Craftsmanship**
{{< /notice >}}

<br>

3. **Debugging Challenges:** Debugging reactive code proves nearly impossible with standard IDE debuggers due to lambdas encapsulating most code.
   Additionally, the loss of meaningful stack traces during exceptions further hampers debugging efforts.
4. **Increased Development and Testing Efforts:** The inherent complexity of reactive code can lead to longer development cycles due to the time required for writing, modifying, and testing.

<br>

Here's an example of reactive code using Mutiny to illustrate the complexity:

```java {linenos=inline}
Multi.createFrom().ticks().every(Duration.ofSeconds(15))
    .onItem().invoke(() - > Multi.createFrom().iterable(configs())
    .onItem().transform(configuration - > {
  try {
    return Tuple2.of(openAPIConfiguration,
        RestClientBuilder.newBuilder()
            .baseUrl(new URL(configuration.url()))
            .build(MyReactiveRestClient.class)
            .getAPIResponse());

  } catch (MalformedURLException e) {
    log.error("Unable to create url");

  }
  return null;
}).collect().asList().toMulti().onItem().transformToMultiAndConcatenate(tuples - > {

  AtomicInteger callbackCount = new AtomicInteger();
  return Multi.createFrom().emitter(emitter - > Multi.createFrom().iterable(tuples)
      .subscribe().with(tuple - >
          tuple.getItem2().subscribe().with(response - > {
              emitter.emit(callbackCount.incrementAndGet());

  if (callbackCount.get() == tuples.size()) {
    emitter.complete();
  }
                    })
                ));

}).subscribe().with(s - > {},
Throwable::printStackTrace, () - > doSomethingUponComplete()))
    .subscribe().with(aLong - > log.info("Tic Tac with iteration: " + aLong));
```

<br>

## Future Outlook-Project Loom and Beyond

Project Loom, a recent development in the Java ecosystem, promises to mitigate the issues associated with blocking operations.
By enabling the creation of thousands of virtual threads without hardware changes, Project Loom could potentially eliminate the need for a reactive approach in many cases.

<br>

{{< notice note >}}
"Project Loom is going to kill Reactive Programming"
**- Brian Goetz**
{{< /notice >}}

<br>

## Conclusion

In conclusion, our decision to move away from reactive architecture style a pragmatic approach to our project's long-term maintainability. While reactive systems offer potential benefits, the challenges they presented for our team outweighed those advantages in our specific context.

Importantly, this shift did not compromise performance. This is a positive outcome, as it demonstrates that a well-designed non-reactive(imperative) architecture can deliver the necessary performance without the complexity associated with reactive architecture in our case.

As we look towards the future, the focus remains on building a codebase that is not only functional but also easy to understand and maintain for developers of all experience levels. This not only reduces development time but also fosters better collaboration and knowledge sharing within the team.

In the graph below, the **X-axis** represents the increasing complexity of our codebase as it evolves, while the **Y-axis** depicts the time required for these developmental changes.

<br>

<img src="/images/reactive-imperative.jpg" alt="Reactive-Imperative" title="Reactive-Imperative">