+++
title = 'Harnessing Automatic Setup and Integration with Quarkus Dev Services for Efficient Development'
date = 2024-06-02T11:04:18+03:00

images = ['images/quarkus_logo_1.png']

tags = ['development', 'quarkus']

categories = ['Quarkus']

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



JPrime 2024 concluded successfully!!

The organizers of JPrime 2024 have once again gone to great lengths to offer a diverse range of
topics, ensuring there's something for everyone.

However, today's article isn't triggered by one of Michael Simons' lectures on **"The Evolution of
Integration Testing within Spring and Quarkus"** although it was highly insightful. He explored
integration testing strategies, focusing on the setup in Spring Boot.

The author clearly emphasized that the issues he highlighted are effectively addressed in Quarkus
through the utilization of Dev Services (**Figure 1**).
This highlights another reason why I view Spring Boot with skepticism for certain applications—its
complexities are starkly contrasted by the streamlined solutions in Quarkus, particularly with the
use of Dev Services.
<br>
<img class="zoom" src="/images/jprime-image.jpg" alt="JPrime" title="JPrime">

_Figure 1 – JPrime 2024_

It was remarkable to witness the astonishment Dev Services sparked among the new attendees. However, it's important to note that Dev Services is not a recent feature in Quarkus; it has been an integral part of the framework for quite some time. Let’s delve deeper into Quarkus Dev Services and explore its enduring benefits.

## Quarkus Dev Services

In Quarkus, Dev Services facilitate the automatic provisioning of unconfigured services in both
development and testing modes. Essentially, if you include an extension without configuring it,
Quarkus will automatically initiate the relevant service- often utilizing Testcontainers in the
background- and configure your application to use this service efficiently.

1. __Automatic Service Detection and Launch__

Quarkus Dev Services automates the detection and launching of necessary services like databases,
message brokers, and other back-end services. It does this by tapping into the application’s
dependencies specified in `pom.xml` or `build.gradle`. For instance, adding a database driver
automatically triggers Dev Services to spin up a corresponding containerized instance of that
database if it's not already running.

The technology used here primarily involves Testcontainers, which allows the creation of
lightweight, throwaway instances of common databases, Selenium web browsers, or anything else that
can run in a Docker container.

2. __Dynamic Configuration Injection__

Once the required services are instantiated, Quarkus Dev Services dynamically injects the relevant
service connection details into the application's configuration at runtime. This is done without any
manual intervention, using a feature known as Continuous Testing that reroutes the standard
database, or other service URLs, to the auto-provisioned Testcontainers. Configuration properties
such as URLs, user credentials, and other operational parameters are seamlessly set, allowing the
application to interact with these services as though they were manually configured

3. __Service-Specific Behaviors__

Dev Services is tailored for various types of services:

* __Databases:__ Automatically provides a running database tailored to your application's needs.
  Whether
  it's PostgreSQL, MySQL, MongoDB, or any other supported database, Dev Services ensures that a
  corresponding Testcontainer is available during development.

* __Messaging Systems:__ For applications that use messaging systems like Kafka or AMQP, Quarkus Dev
  Services starts the necessary brokers, again using Docker, and connects them with the application.

* __Custom Dev Services:__ Developers can extend the functionality by creating custom Quarkus
  extensions that leverage the Dev Services framework. This allows for tailored setups that are
  project-specific, offering even greater flexibility and control.


4. __Network Handling and Service Isolation__

Each service spun up by Quarkus Dev Services runs in its isolated environment. This is crucial for
ensuring that there are no port conflicts, data residue, or security issues between different
development tests. Despite this isolation, services are networked appropriately using Docker,
ensuring that they can communicate with each other as needed, imitating a real-world deployment
atmosphere.

5. __Lifecycle Management__

Quarkus manages the complete lifecycle of these dynamically provisioned services. When you start
your application in development mode, the necessary services are started up automatically. When you
stop the Quarkus application, these services are also terminated. This management includes handling
data persistency as required, allowing developers to pick up right where they left off without any
setup delays.

## Example Usage

Consider you’re using a PostgreSQL database with Quarkus. If no existing PostgreSQL configuration is
detected, Quarkus will kickstart a PostgreSQL Docker container and connect your application
automatically.

These services are enabled by default in development and test modes but can be disabled if necessary
via the `application.properties`:

``` properties
quarkus.datasource.devservices.enabled=false
```

Let's expand on the scenario where Quarkus is using a PostgreSQL database and how the Dev Services
facilitate this with minimum fuss.

If Quarkus detects that no PostgreSQL configuration is active (not running or not configured
explicitly), it will automatically start up a PostgreSQL container using Docker. This is set up
behind the scenes through Dev Services.

To interact with the database through an ORM layer, consider using Quarkus Panache, which simplifies
Hibernate ORM operations. Here’s how to set up your environment:

1. Add Dependencies

Firstly, include the necessary dependencies in your `pom.xml`:

``` xml {linenos=inline}
<dependency>
   <groupId>io.quarkus</groupId>
   <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
<dependency>
   <groupId>io.quarkus</groupId>
   <artifactId>quarkus-jdbc-postgresql</artifactId>
</dependency>
```

2. Define the Entity

Next, define your entity, such as `CityEntity`:

``` java {linenos=inline}
@Entity
@Table(name = "cities")
public class CityEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;

  @Column(name = "public_id")
  private String publicId;

  @OneToOne
  private StateEntity state;

  @Column(nullable = false, name = "created_at")
  private Instant createdAt;

  @Column(nullable = false, name = "last_modified")
  private Instant lastModified;

  @PrePersist
  protected void onCreate() {
    createdAt = Instant.now();
    lastModified = createdAt;
  }

  @PreUpdate
  protected void onUpdate() {
    lastModified = Instant.now();
  }
}
```

3. Create the Repository

Implement the repository which will directly interact with the database:

``` java {linenos=inline}
@ApplicationScoped
public class CityRepository implements PanacheRepository<CityEntity> {
}
```

4. Service Layer

Define the service layer that utilizes the repository:

``` java {linenos=inline}
@ApplicationScoped
public class CityServiceImpl implements CityService {

  @Inject
  CityRepository cityRepository;

  @Override
  public long countCities() {
    return cityRepository.count();
  }
}

public interface CityService {
  long countCities();
}
```

5. Resource Endpoint

``` java {linenos=inline}
@Path("/cities")
@Tag(name = "City Resource", description = "City APIs")
public class CityResource {

  @Inject
  CityService cityService;

  @GET
  @Path("/count")
  @Operation(summary = "Get the total number of cities", description = "Returns the total count of cities in the system.")
  @APIResponse(responseCode = "200", description = "Successful response", content = @Content(mediaType = "application/json", schema = @Schema(implementation = Long.class)))
  public long count() {
    return cityService.countCities();
  }
}

````

When you run your Quarkus application (`mvn quarkus:dev`), observe the automatic startup of the
PostgreSQL container (**Figure 2**). This seamless integration exemplifies the power of Quarkus
Dev Services, making development and testing significantly simpler by automating the configuration
and connection setup to external services needed for your application.

<br>
<img class="zoom" src="/images/quarkus-dev-services-1.jpg" alt="Application logs" title="Application logs">

_Figure 2 – Application logs_

## Platform Dev Services

Quarkus Dev Services streamline the development and testing phases by handling the configuration and
management of various services, allowing developers to focus more on the actual application. Quarkus
supports a wide range of Dev Services, including:

- AMQP
- Apicurio Registry
- Databases
- Kafka
- Keycloak
- Kubernetes
- MongoDB
- RabbitMQ
- Pulsar
- Redis
- Vault
- Infinispan
- Elasticsearch
- Observability
- Neo4j
- WireMock
- Microcks
- Keycloak
- and many more, each designed to enhance your development environment seamlessly

## Conclusion

Quarkus Dev Services represents a paradigm shift in how developers approach setting up and
integrating external services during the development and testing phases. The automation of
environment setup not only accelerates the development process but also reduces the potential for
configuration errors, making it easier for teams to focus on creating robust, feature-rich
applications.

One of the standout advantages of Quarkus Dev Services is the emphasis on developer productivity. By
removing the need to manually manage service dependencies, developers can immediately begin work on
business logic and application features. This streamlined workflow is particularly beneficial in
microservices architectures where multiple services might require simultaneous development and
integration

In conclusion, embracing Quarkus Dev Services could significantly impact your development team's
effectiveness and project outcomes. The simplicity and power of Quarkus encourage experimentation,
quicker iterations, and ultimately a faster development cycle. This kind of technological leverage
is what modern businesses need to thrive in the digital era.