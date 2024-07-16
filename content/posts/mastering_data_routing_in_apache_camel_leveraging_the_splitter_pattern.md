+++
title = 'Mastering Data Routing in Apache Camel: Leveraging the Splitter Pattern'
date = 2024-07-15

images = ['images/apache_camel_12.png']

tags = ['etl', 'camel']

categories = ['Apache Camel']

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


Hello again! In my upcoming articles, I plan to explore several key patterns provided by Apache Camel that are invaluable for your ETL processes. 
This current article focuses on the Splitter Enterprise Integration Pattern (EIP).

# Overview

Messages passed to integration applications often arrive in a format that is less than ideal for immediate processing. Frequently, these are 
composite messages containing multiple elements, each requiring individual processing. This is where the Splitter pattern proves beneficial by 
dividing incoming messages into a sequence of more manageable messages.

The diagram below (**Figure 1**) illustrates the simplicity of the Splitter pattern- it takes a single message and splits it into multiple distinct messages.

<br>

<img class="zoom" src="/images/spliter-pattern-ac.png" alt="Splitter pattern" title="Splitter pattern">

_Figure 1 – Splitter Pattern (enterpriseintegrationpatterns.com)_

<br>

# Using Splitter in Apache Camel

Implementing the Splitter pattern in Apache Camel is straightforward. Incorporate the split method in the route definition. This method requires an 
Expression to dictate how the message should be split.

Commonly, you might need to split a **list**, **set**, **map**, **array**, or any other **collection**. For such cases, use the body method to 
retrieve the message body before splitting.

The Splitter behaves akin to a comprehensive iterator that processes each entry consecutively. The sequence diagram in (**Figure 2**) offers detailed 
insights into this iterator's operation.

<br>

<img class="zoom" src="/images/split-diagrama.jpg" alt="A sequence diagram" title="A sequence diagram">

_Figure 2 – A sequence diagram_

Upon receiving a message, the Splitter evaluates an Expression which returns the message body. This result is then utilized to create a `java.util.Iterator`.

By default, the split method divides the message body based on its value type, as illustrated below:

- `java.util.Collection`: Splits each element from the collection.

- `java.util.Map`: Splits each map entry.

- `Object[]`: Splits each array element.

- `Iterator`: Splits each iteration.

- `Iterable`: Splits each iterable component.

- `String`: Splits each string character.

The Splitter then utilizes the iterator till all data is processed. Each resultant message is a copy of the original with its body replaced by 
the portion from the iterator.

Let's explore how to employ the Splitter pattern in Apache Camel with practical examples.

**Example 1:** Splitting an `ArrayList` of `Strings`

Consider a route that splits an `ArrayList` of `String` and logs each element. Here's how you define this route:

```java {linenos=inline}
@ApplicationScoped
public class SplitterRoute extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("direct:split_start_1")
        .log("Before Split line ${body}")
        .split(body())
        .log("Split line ${body}");
    }
}
```

Call the above route by passing an `ArrayList` of `Strings` as the message body:

```java {linenos=inline}
List<String> body = List.of("A", "B", "C");
producerTemplate.sendBody("direct:split_start_1", body);
```

Running the above code yields the following output:

```shell {linenos=inline}
2023-07-14 20:10:03,389 INFO  Before Split line [A, B, C]
2023-07-14 20:10:03,391 INFO  Split line A
2023-07-14 20:10:03,392 INFO  Split line B
2023-07-14 20:10:03,392 INFO  Split line C
```
<br>

**Example 2:** Splitting a `String` by Custom Delimiter

In this example, we split a String using a custom delimiter `#`:

```java {linenos=inline}
@ApplicationScoped
public class SplitterRoute extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("direct:split_start_2")
        .log("Before Split line ${body}")
        .split(body(), "#")
        .log("Split line ${body}");
    }
}
```

To call the route and pass the `String` as a message body:

```java {linenos=inline}
producerTemplate.sendBody("direct:split_start_2", "A#B#C");
```

Running this code, you see the following in the log:

```shell  {linenos=inline} 
2024-07-14 20:19:19,229 INFO  Before Split line A#B#C
2024-07-14 20:19:19,229 INFO  Split line A
2024-07-14 20:19:19,229 INFO  Split line B
2024-07-14 20:19:19,230 INFO  Split line C
```

These examples demonstrate the versatility of the Splitter pattern in handling various types of message formats. 

As you can see, the Splitter pattern is very useful when you need to split a message into multiple messages. But let's see another example
where we are going to split an `Object` by its fields. For the demo purpose, we are going to  provide the `User` object with the `List` of `Address` objects.

The `User` class is:

```java {linenos=inline}
public record UserDetails(String userId, List<Address> addresses) {}
```

and the `Address` class is:

```java {linenos=inline}
public record Address(String addressId, String address) {}
```

The route definition is:

```java {linenos=inline}
@ApplicationScoped
public class SplitterRoute extends RouteBuilder {

  @Override
  public void configure() throws Exception {
    from("direct:userDetails")
        .log("User Id: ${body.userId}")
        .split(method(UserService.class))
        .log("Split line ${body}");
  }

  static class UserService {
    public static List<Address> getDetails(UserDetails userDetails) {
      return userDetails.addresses();
    }
  }
}
```

and to call the above route as pass the `User` object as a message body:

```java {linenos=inline}
final UserDetails userDetails = new UserDetails(
    "user-1",
    List.of(
        new Address("1", "Address 1"),
        new Address("2", "Address 2")
    )
);

producerTemplate.sendBody("direct:userDetails", userDetails);
```

When you run the above code, you will see the following output in the log:

```shell
2024-07-14 21:09:02,225 INFO  [route36] (Quarkus Main Thread) User Id: user-1
2024-07-14 21:09:02,226 INFO  [route36] (Quarkus Main Thread) Split line Address[addressId=1, address=Address 1]
2024-07-14 21:09:02,226 INFO  [route36] (Quarkus Main Thread) Split line Address[addressId=2, address=Address 2]
```
The method component in Apache Camel is a versatile and powerful tool that allows routes to directly invoke methods on beans. 
This capability is particularly useful in enterprise applications where business logic needs to be cleanly encapsulated within service 
classes, yet seamlessly integrated within Camel routes for orchestration of data flows.

The route invokes a method from the `UserService` class. Apache Camel looks for this method in the `UserService` class to handle how the 
message should be split. The method should be designed to accept the incoming message body and return a collection, such as a `List` or `Array`, 
which Camel then iterates over. Each element of the array or list becomes a new exchange in Camel, which is processed independently for 
the remainder of the route.

<br>

By integrating dynamic splitting with conditional routing using `choice()` and `when()` constructs, Apache Camel provides a robust framework for 
bespoke data processing workflows.


## Understanding `choice()` and `when()` in Apache Camel

In Apache Camel, `choice()` and when are powerful constructs that enable content-based routing. This paradigm mimics the `if-else` or `switch-case`
constructs familiar in many programming languages, allowing routes to make decisions based on the content of messages.

The `choice()` construct serves as a container for one or more `when()` clauses. Each `when()` clause specifies a condition. When a condition evaluates to
true, Apache Camel routes the message to the processing logic defined within that `when()` block. If none of the conditions in any of the `when()` clauses
match, an optional `otherwise()` clause can be executed, akin to an else statement in traditional programming.

Detailed Example with `choice()` and `when()`.

Imagine a scenario where an application processes orders and sends them to different processing streams based on their category, such as "books", 
"electronics", or "clothing". This setup allows for highly specialized processing for different categories of items.

Here's how you might set up such a route in Apache Camel:

- **Route Configuration:** Define a route that starts from an endpoint (for example, `direct:startOrderRoute`). This route will examine the type of order and route the message accordingly.

- **Using `choice()` `and when()`:** Inside the route, a `choice()` component examines the message. Based on the content, it routes the message to different endpoints using `when()`.

- **Logging and Processing:** Each category has a dedicated log that records the action taken, ensuring traceability.

<br>

Here's a sample implementation:

```java {linenos=inline}
public class OrderRouteBuilder extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("direct:start")
            .choice()
                .when(body().contains("books"))
                    .log("Routing to Books processing: ${body}")
                    .to("direct:books")
                .when(body().contains("electronics"))
                    .log("Routing to Electronics processing: ${body}")
                    .to("direct:electronics")
                .when(body().contains("clothing"))
                    .log("Routing to Clothing processing: ${body}")
                    .to("direct:clothing")
                .otherwise()
                    .log("Unknown order type: ${body}")
            .end();

        from("direct:books")
            .log("Processing Books Order: ${body}");

        from("direct:electronics")
            .log("Processing Electronics Order: ${body}");

        from("direct:clothing")
            .log("Processing Clothing Order: ${body}");
    }
}
```

Now you can run the route by sending different order types to the `direct:start` endpoint:

```java {linenos=inline}
producerTemplate.sendBody("direct:startOrderRoute", "An order for books");
```
and you will see the following output in the log:

```shell {linenos=inline}
2024-07-16 19:16:12,357 INFO  [route5] (Quarkus Main Thread) Received Order: An order for books
2024-07-16 19:16:12,357 INFO  [route5] (Quarkus Main Thread) Routing to Books processing: An order for books
2024-07-16 19:16:12,358 INFO  [route6] (Quarkus Main Thread) Processing Books Order: An order for books
```
As you can see, the message is routed to the `books` endpoint and processed accordingly. You can test the other categories in a similar manner.

<br>

### Practical Usage and Tips

- **Complex Conditions:** `when()` can handle complex expressions combining multiple conditions.

- **Using `.endChoice()`:** For readability and to prevent misrouting, especially in a route with nested `.choice()` constructs, use `.endChoice()` to clearly indicate the end of a choice block.

- **Flexibility:** This approach is highly flexible, allowing the addition of new conditions and endpoints as business requirements evolve.

### Conclusion

The Splitter pattern stands out as a powerful tool in Apache Camel's arsenal, enabling developers to effectively handle complex, bulky data structures by breaking them down into more manageable pieces. This pattern not only simplifies data processing workflows but also enhances maintainability and scalability within integration solutions.

In this article, we've explored the practical implementation of the Splitter pattern in a range of contexts - from handling lists and custom delimited strings to operating on more complex structures like Java objects. Each example showcased how Apache Camel facilitates straightforward data manipulations, making seemingly complex integration tasks much simpler.

Additionally, the integration of `choice()` and `when()` constructs further refines the data routing capabilities in Apache Camel, offering precise control over message flow based on specific content criteria. This ability to route data conditionally mimics conventional programming logic, bringing a familiar and intuitive approach to route definitions.

As businesses continue to deal with increasingly diverse and voluminous data, understanding and implementing these patterns will become crucial. The flexibility and power of Apache Camel in managing data flows allow developers to build robust, efficient data ingestion and processing pipelines that are crucial for today's data-driven applications.

And last but not least, the all above examples are implemented in the Quarkus framework as using the Camel Quarkus extension.