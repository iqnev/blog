+++
title = 'Extending Quarkus: When and How to Write Your Own Extensions'
date = 2023-10-08T21:29:18+03:00

tags = ['Development', 'Quarkus', 'Extension']

categories = ['quarkus']

noComment= false
+++


Quarkus, with its innovative extension framework, offers developers a powerful way to integrate
various technologies seamlessly into their applications.
These extensions simplify configuration, enable dependency injection, and optimize performance,
making it an attractive option for Java developers.
However, before diving into creating your own Quarkus extension, it's crucial to understand when it'
s necessary and how to do it effectively.

## When to Create a Quarkus Extension

1. Complex Integrations: If you're working with complex frameworks like ORM mappers, reactive
   clients, or data access libraries, creating an extension can help manage the intricacies of
   configuration and dependency management.
   Extensions simplify the use of these frameworks in Quarkus applications.

2. Performance Optimization: Quarkus extensions are designed to align with Quarkus' native
   compilation, resulting in applications that start swiftly and have minimal memory footprints.
   By creating an extension, you can leverage Quarkus' build-time optimization abilities to scan
   dependencies and generate configuration early, thus avoiding startup delays.

3. Developer Experience Enhancement: Extensions can significantly enhance the developer experience.
   They enable live reloading, CLI extensions, templating, and more, streamlining the development
   process.
   If you want to provide a seamless and efficient development environment for your team, extensions
   can help achieve this goal.

4. API Hardening: If you're building APIs or libraries intended to be used by other Quarkus
   developers, extensions provide an excellent way to harden your
   APIs and ensure they work seamlessly within the Quarkus ecosystem.

However, extensions may not always be the best approach. For simpler needs, such as sharing utility
code and glue logic between components, a basic JAR file might
suffice without the overhead of creating an extension. If your integration is app-specific and
unlikely to be reused elsewhere, a basic JAR could be a more straightforward solution. Moreover, if
you need full control over dependency versions and don't want to adhere to Quarkus' BOM (Bill of
Materials) for dependency management, a JAR may be a better choice. Finally, if your code needs to
work across multiple JVM frameworks, such as Spring and Micronaut,
avoiding tight coupling to Quarkus may be preferable.

Creating Quarkus extensions can be complex, often requiring in-depth knowledge of Quarkus internal
workings. However, for many scenarios, creating a standard JAR can be sufficient. This JAR, when
indexed by Jandex, can be seamlessly discovered by Quarkus during build time. While Quarkus
extensions provide a range of advantages, including superior performance and developer productivity,
they may not always be necessary.

Quarkus unique approach to moving work to build time, rather than runtime, is at the core of its
fast startup times and low memory footprint. This philosophy extends to Quarkus extensions, which
can leverage these build-time optimizations. Even if you're not primarily concerned with fast boot
times, the benefits of creating your extensions extend to simplifying configurations, extending the
Quarkus CLI, and integrating with Quarkus's Dev Mode.

Creating your Quarkus extensions doesn't have to be overly complicated. With the right approach and
a clear understanding of your project's needs, you can solve complex problems efficiently.
Extensions offer a flexible and powerful way to enhance your Quarkus applications and make them more
efficient and developer-friendly.

## Creating a Quarkus Extension

When you decide that creating a Quarkus extension is the right approach, it's essential to
understand the structural components of an extension:

* **Runtime Section:** This section contains the core business logic implemented as beans, services,
  or other components that integrate with Quarkus;
* **Deployment Section:** The deployment section handles build-time augmentation and configuration.
  It ensures that your extension integrates seamlessly with Quarkus' optimization processes;
* **Descriptor:** A descriptor declares metadata about your extension, including its name,
  parameters, compatibility information, and more;
* **Documentation:** Comprehensive documentation should accompany your extension. It guides users on
  how to use and configure your extension effectively.

## Anatomy of the Quarkus Extension

Consider a scenario where you want to create a custom caching extension for Quarkus. This extension
will allow developers to easily integrate caching functionality into their Quarkus applications.

1. **Runtime Section:**
    - In this section, you would implement the core caching functionality using Java code. This
      might include methods for caching data, retrieving cached data, and managing cache expiration.
    - For example, you might have a `CustomCacheService` class with methods
      like `put(key, value)`, `get(key)`, and `evict(key)` to handle caching operations.

2. **Deployment Section:**
    - The deployment section is responsible for build-time optimization. Here, you can specify how
      the caching configuration should be generated during the build process.
    - For our caching extension, this section might include instructions on how to scan for cached
      objects in the application code and generate cache configuration.

3. **Descriptor:**
    - The descriptor file (`custom-cache-extension.yaml`) provides metadata about your extension. It
      includes information like the extension's name, version, compatibility with Quarkus, and
      configuration parameters.
    - For instance, your descriptor might specify that the extension is named "
      custom-cache-extension," is compatible with Quarkus 2.0+, and requires a cache timeout
      configuration parameter.

4. **Documentation:**
    - Comprehensive documentation should accompany your extension. It guides users on how to use the
      custom caching extension effectively within their Quarkus applications.
    - Documentation should include examples of how to configure the cache, integrate it into Quarkus
      services, and manage cached data. Additionally, it should provide best practices for cache
      utilization.

By following this structure, your custom caching extension becomes a valuable tool for Quarkus
developers. They can easily incorporate caching into their applications, improving performance and
optimizing resource usage.

Runtime module:

```java
class CustomCacheService {
  
    // Core caching functionality using Java code
    public void put(String key, Object value) {
      // Cache data implementation
    }
    
    public Object get(String key) {
      // Retrieve cached data implementation
    }
    
    public void evict(String key) {
      // Evict cached data implementation
    }
}
```

Deployment module:

```java
class CustomCacheProcessor {
    @BuildStep
    FeatureBuildItem feature() {
        // This declares the custom cache extension as a feature
        return new FeatureBuildItem("custom-cache");
    }
}
```

Descriptor file: `custom-cache-extension.yaml`

```yaml
extension:
name: custom-cache-extension
metadata:
    short-name: "resteasy-reactive"
    keywords:
    - "jaxrs"
    - "web"
    - "rest"
    categories:
    - "web"
    - "reactive"
    status: "stable"
    guide: "https://quarkus.io/guides/resteasy-reactive"
```

In conclusion, whether to create a Quarkus extension depends on your project's specific needs and
objectives.
Quarkus extensions are powerful tools for deep integration, performance optimization, and enhancing
the developer experience.
However, it's essential to weigh the trade-offs and consider whether a simpler solution, like a
standard JAR library, might better suit your use case.
By understanding when and how to create Quarkus extensions effectively, you can make informed
decisions and leverage the full potential of this innovative framework.
