+++
title = 'Practical Guide to Apache Camel with Quarkus: Building an ETL Application'
date = 2024-06-09

images = ['images/apache-camel-logo-1.png']

tags = ['quarkus', 'etl', 'camel']

categories = ['Quarkus', 'Apache Camel']

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

I am excited to introduce a series of articles about Apache Camel. In this first post, rather than delving deeply into the complexities of Apache Camel, 
I will present a practical use case to showcase its capabilities. Specifically, you'll learn how to create a simple Extract, Transform, and Load (ETL) 
application between two databases using Apache Camel.

## Introduction to Apache Camel - Brief Overview

Before we dive into the practical use case, let's briefly introduce Apache Camel. Apache Camel is an open-source integration framework that leverages Enterprise Integration Patterns (EIP) to 
facilitate the integration of various systems.

In today's world, numerous systems of different types coexist. Some may be legacy systems, while others are new. These systems often need to interact and integrate with each other, which can be 
challenging due to differing implementations and message formats. One solution is to write custom code to bridge these differences, but this can lead to tight coupling and maintenance difficulties.

Instead, Apache Camel offers an additional layer to mediate the differences between systems, resulting in loose coupling and easier maintenance. Camel uses an API (or declarative Java Domain Specific Language) 
to configure routing and mediation rules based on EIP.

### Enterprise Integration Patterns (EIP)

To understand Apache Camel, it's important to grasp "Enterprise Integration Patterns" (EIP). The book "Enterprise Integration Patterns" describes a set of patterns for designing large, component-based systems where 
components can run in the same process or on different machines. The key idea is that systems should be message-oriented, with components communicating via messages. The patterns provide a toolkit for implementing 
these communications (**Figure 1**).

<br>

<img class="zoom" src="/images/basic_lements_of_an_integration_solution.jpg" alt="Basic Elements of an Integration Solution" title="Basic Elements of an Integration Solution">

_Figure 1 – Basic Elements of an Integration Solution (enterpriseintegrationpatterns.com)_


## Key Terminologies in Apache Camel

- **EndPoint:** An endpoint is a channel through which messages are sent and received. It serves as the interface between a component and the outside world.

- **Message:** A message is a data structure used for communication between systems, consisting of a header and a body. The header contains metadata, and the body contains the actual data.

- **Channel:** A channel connects two endpoints, facilitating the sending and receiving of messages.

- **Router:** A router directs messages from one endpoint to another, determining the message path.

- **Translator:** A translator converts messages from one format to another.

I consider to stop here for now about the introduction to Apache Camel. Now let's show you how to create a simple ETL application between two databases using Apache Camel.

### Problem Statement

Let's assume we have a highly loaded system where one critical component is the database. At some point, we need to deal with this data outside of the usual operational 
cases - training ML models, generating exports, graphs, or we simply need some part of the data. Of course, this would burden our operational database even more, and for 
this purpose, it would be optimal to have a mechanism with which we can extract the necessary data, transform it into the form we need, and store it in another database - other 
than the operational one. With this strategy, we solve the problems of excessive potential overloading of our operational base. Moreover, with such a mechanism, we can perform
this operation at a time when our system is not too loaded (e.g., nighttime).

## Solution Overview

The solution is shown in the diagram below (**Figure 2**). We will use Apache Camel to create a simple ETL application between two databases. The application will extract data from a source 
database, transform it, and load it into a target database. We can introduce different strategies to implement this solution, focusing on how to extract the data from the 
source database. I assume that the criteria for selecting the data will be based on the modification date of the record. This strategy provides an opportunity to extract 
the data that has been modified, too.

<br>

<img class="zoom" src="/images/apache-camel-syncer.jpg" alt="Syncing Data Between Two Databases Using Apache Camel" title="Syncing Data Between Two Databases Using Apache Camel">

_Figure 2 – Syncing Data Between Two Databases Using Apache Camel_


Source and target databases will have the following table structure:

```sql {linenos=inline}
CREATE TABLE IF NOT EXISTS user
(
    id            serial PRIMARY KEY,
    username      VARCHAR(50)  NOT NULL,
    password      VARCHAR(50)  NOT NULL,
    email         VARCHAR(255) NOT NULL,
    created_at    timestamp default now()::timestamp without time zone,
    last_modified TIMESTAMP DEFAULT now()::timestamp without time zone
);
```

In the target database, we'll transform the username to uppercase before inserting it.

We'll use Camel Quarkus extensions for various Camel components. Specifically, we'll use the Camel SQL component to interact with the databases.
The SQL component supports performing SQL queries, inserts, updates, and deletes.

First, create a class that extends `RouteBuilder` and override the `configure` method:

```java {linenos=inline}
@ApplicationScoped
public class UserRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        // your code here
    }
}
```
The used of `@ApplicationScoped` annotation is not mandatory here, but I prefer to indicate that the class is a CDI bean and should be managed by the CDI container.

As I mentioned above, we will use the Camel SQL component to interact with the databases. We need to configure the Camel SQL component to connect 
to the source and target databases. We will use the Quarkus Agroal extension to configure the data sources. The Agroal extension provides a 
connection pool for the data sources. We will configure the data sources in the `application.properties` file.

```properties {linenos=inline}
#
# Source Database Configuration
quarkus.datasource.source_db.db-kind=postgresql
quarkus.datasource.source_db.jdbc.url=jdbc:postgresql://localhost:5001/demo
quarkus.datasource.source_db.username=test
quarkus.datasource.source_db.password=password1
#
#
# Target Database Configuration
quarkus.datasource.target_db.db-kind=postgresql
quarkus.datasource.target_db.jdbc.url=jdbc:postgresql://localhost:6001/demo
quarkus.datasource.target_db.username=test
quarkus.datasource.target_db.password=password1
#
```

Now we can configure the Camel SQL component to connect to the source and target databases. We will use the `sql` component to create a SQL endpoint for the source and 
target databases.

The SQL component uses the following endpoint URI notation:

```sql {linenos=inline}
sql:select * from table where id=# order by name[?options]
```

But we need mechanism to run the operation automatically. We will use the `timer` component to trigger the ETL process every seconds. The `timer` component is used to 
generate message exchanges when a timer fires. The timer component uses the following endpoint URI notation:

```text {linenos=inline}
timer:name[?options]
```

In our route we use configuration as follows:

```java {linenos=inline}
 from("timer://userSync?delay={{etl.timer.delay}}&period={{etl.timer.period}}")
```

The `{{etl.timer.delay}}` and `{{etl.timer.period}}` are the configuration values that we will define in the `application.properties` file.

```properties {linenos=inline}
etl.timer.period=10000
etl.timer.delay=1000
```

In order to transform the data before inserting it into the target database, we need to provide our translator:

```java {linenos=inline}
.process(exchange -> {
    final Map<String, Object> rows = exchange.getIn().getBody(Map.class);

    final String userName = (String) rows.get("username");

    final String userNameToUpperCase = userName.toUpperCase();

    log.info("User name: {} converted to upper case: {}", userName, userNameToUpperCase);

    rows.put("username", userNameToUpperCase);
})

```

The `Processor` interface is used to implement consumers of message exchanges or to implement a Message Translator and other use-cases.


And voila, we have a simple ETL application between two databases using Apache Camel. 

When you run the application, you should see the following output in the logs:

```text
2024-06-09 13:15:49,257 INFO  [route1] (Camel (camel-1) thread #1 - timer://userSync) Extracting Max last_modified value from source database
2024-06-09 13:15:49,258 INFO  [route1] (Camel (camel-1) thread #1 - timer://userSync) No record found in target database
2024-06-09 13:15:49,258 INFO  [route2] (Camel (camel-1) thread #1 - timer://userSync) The last_modified from source DB: 
2024-06-09 13:15:49,274 INFO  [route2] (Camel (camel-1) thread #1 - timer://userSync) Extracting records from source database
2024-06-09 13:15:49,277 INFO  [org.iqn.cam.rou.UserRoute] (Camel (camel-1) thread #1 - timer://userSync) User name: john_doe converted to upper case: JOHN_DOE
2024-06-09 13:15:49,282 INFO  [org.iqn.cam.rou.UserRoute] (Camel (camel-1) thread #1 - timer://userSync) User name: jane_smith converted to upper case: JANE_SMITH
2024-06-09 13:15:49,283 INFO  [org.iqn.cam.rou.UserRoute] (Camel (camel-1) thread #1 - timer://userSync) User name: alice_miller converted to upper case: ALICE_MILLER

```

You can find the full source code of the application in the [GitHub repository](https://github.com/iqnev/quarkus-camel-sync-db).

## Conclusion
With this setup, we've created a simple ETL application using Apache Camel that extracts data from a source database, transforms it, and loads it into a target database. 
This approach helps reduce the load on the operational database and allows us to perform data extraction during off-peak times.
