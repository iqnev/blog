+++
title = 'Unveiling Challenges with @Named'
date = 2023-12-09T11:04:18+03:00

images = ['images/java-ee.png']

tags = ['cdi', 'dependency-injection', 'java-ee']

categories = ['Quarkus']

noComment = false
+++

In the ever-evolving landscape of Contexts and Dependency Injection (**CDI**), developers frequently
encounter hurdles related to bean naming, default implementations, and potential conflicts. This
article provides a detailed exploration of the potential pitfalls associated with the `@Named`
annotation in **CDI**. We will delve into its intricacies, shed light on problematic scenarios, and
discuss alternative approaches, including the use of `@Identifier` from **SmallRye**. Furthermore,
we'll offer insights into best practices for building robust and maintainable **Jakarta EE**
applications.

## Understanding `@Default`

The `@Default` annotation is a valuable tool in **CDI** for explicitly marking a specific
implementation
as the default one for a given interface or bean type. It comes into play when dealing with multiple
implementations of the same interface, allowing developers to specify which implementation should be
injected by default when no other qualifiers are used.

Consider a scenario where multiple implementations of the `GreetingService` interface exist:

```java {linenos=inline}

@Default
public class DefaultGreetingService implements GreetingService {

  @Override
  public String greet(String name) {
    return "Hello, " + name;
  }
}
```

```java {linenos=inline}
public class SpecialGreetingService implements GreetingService {

  @Override
  public String greet(String name) {
    return "Greetings, " + name + "!";
  }
}
```

When injecting a bean without specifying any qualifiers, **CDI** uses the `@Default` -marked bean as
the
default. This is beneficial in scenarios with multiple implementations, providing a clear default
choice.

```java {linenos=inline}

@Inject
private GreetingService greetingService; // Injects the @Default implementation
```

While the use of `@Default` is optional, it's highly recommended, particularly when dealing with
interfaces that have multiple implementations. It provides a clear and consistent default option,
preventing ambiguity and unexpected behavior during bean injection.

## Exploring `@Named` - a double-edged sword

The `@Named` qualifier plays a fundamental role in **CDI**, assigning a human-readable name or
identifier to a bean. Developers often employ it to refer to beans by name when injecting them into
other components.

However, `@Named` comes with its own set of challenges, particularly when used without additional
qualifiers. By default, **CDI** associates the unqualified class name as the bean name. This can
lead to conflicts with the @Default qualifier, resulting in unexpected behavior during bean
injection.

```java {linenos=inline}

@Named
public class MyBean {
  // Implementation
}

```

When injecting `MyBean` without explicit qualifiers, CDI will only add the `@Named` qualifier, not
the `@Default` qualifier. The `@Default` qualifier is only applied if it is explicitly specified on
the bean or its qualifiers.

```java {linenos=inline}

@Inject
private MyBean myBean;
```

In this case, ambiguity may arise if there are other beans with the same type name. For instance, if
there is another bean named `MyBean`, the injection will result in ambiguity.

To address this issue, developers should explicitly qualify the bean they intend to inject.

```java {linenos=inline}

@Inject
@Named("myBean")
private MyBean myBean;
```

Alternatively, developers can utilize a custom qualifier for each bean to eliminate ambiguity.

## Problematic Cases: Ambiguity and Unintended Defaults

Ambiguity arises when `@Named` is used without additional qualifiers, and multiple implementations
of
the same type exist. Consider the following scenario:

```java {linenos=inline}

@Named
public class ServiceA implements Service {
  // Implementation
}
```

```java {linenos=inline}

@Named
public class ServiceB implements Service {
  // Implementation
}
```

Injecting `Service` without explicit qualifiers can lead to ambiguity since both beans match by
type,
and no name or qualifier distinguishes them.

```java {linenos=inline}

@Inject
private Service service;
```

In this case, **CDI** does not implicitly add `@Default` or attempt to resolve the ambiguity,
resulting in
a failed injection due to an ambiguous dependency.

## Alternatives: Introducing `@Identifier` from SmallRye Common

Acknowledging the challenges posed by `@Named`, developers often seek alternatives for more explicit
control over bean identification. One such alternative is the `@Identifier` annotation
from <a href="https://javadoc.io/doc/io.smallrye.common/smallrye-common-annotation/latest/io/smallrye/common/annotation/Identifier.html">
SmallRye
Common</a> . This annotation offers a clearer and more controlled approach to naming beans, reducing
the
risk of conflicts and unexpected defaults.
In contrast to `@Named`, which requires unique values for each application, `@Identifier` allows for
multiple beans with the same identifier value as long as their types differ. This flexibility is
particularly useful when handling different implementations of the same interface or related types.

To use `@Identifier`, simply annotate the bean class with the annotation and specify the identifier
value:

```java {linenos=inline}

@Identifier("payment")
public class DefaultPaymentProcessor implements PaymentProcessor {
  // Implementation
}
```

```java {linenos=inline}

@Identifier("payment")
public class LegacyPaymentGateway implements PaymentGateway {
  // Implementation
}
```

Injecting beans using `@Identifier` is straightforward:

```java {linenos=inline}
public class Client {

  @Inject
  @Identifier("payment")
  PaymentProcessor processor;

  @Inject
  @Identifier("payment")
  PaymentGateway gateway;

}
```

Here, the "payment" `@Identifier` value is reused for multiple beans because the types
`PaymentProcessor` and `PaymentGateway` differ. This flexibility is not allowed by `@Named`, where
values
must be unique application-wide.

Another alternative to `@Named` is to create custom qualifiers. Custom qualifiers are user-defined
annotations that can be used to identify and qualify beans. They offer the most granular control
over bean selection and can be tailored to specific needs of the application.

To create a custom qualifier, follow these steps:

1. Define a new annotation class.
2. Annotate the annotation class with `@Qualifier`.
3. Optionally, provide a default value for the qualifier.

For example, the following custom qualifier named `DefaultPaymentGateway` indicates the default
payment gateway implementation:

```java {linenos=inline}

@Qualifier
@Retention(RUNTIME)
@Target({METHOD, FIELD, PARAMETER, TYPE})
public @interface DefaultPaymentGateway {

}
```

To use the custom qualifier, annotate the bean class with it:

```java {linenos=inline}

@DefaultPaymentGateway
public class StandardPaymentGateway implements PaymentGateway {
  // Implementation
}
```

```java {linenos=inline}
public class ExpressPaymentGateway implements PaymentGateway {
  // Implementation
}
```

Then, inject the bean using the qualifier:

```java {linenos=inline}

@Inject
@DefaultPaymentGateway
private PaymentGateway paymentGateway;
```

## Choosing the Right Approach

The best approach for bean identification depends on the specific needs of the application. For
simple applications, `@Named` may be sufficient. For more complex applications, `@Identifier` or
custom
qualifiers offer more control and flexibility.

The following table summarizes the pros and cons of each approach:


<style>
table {
    border-collapse: collapse;
    width: 100%;
}

th, td {
    border: 1px solid #dddddd;
    text-align: left;
    padding: 8px;
}
thead {
    background-color: #f0e8e8; /* Change this to the desired background color for the header */
}
</style>


| Approach          | Pros                                                 | Cons                                           |
|-------------------|------------------------------------------------------|------------------------------------------------|
| `@Named`          | Simple, widely supported                             | Can be ambiguous, conflicts with `@Default`    |
| `@Identifier`     | Clearer identification, no conflicts with `@Default` | Requires additional annotations                |
| Custom qualifiers | Maximum flexibility, fine-grained control            | Requires upfront effort to define and maintain |



For further confirmation, you can refer to the official **CDI** specification

<img src="/images/2.2.9.png" alt="2.2.9. The qualifier @Named at injection points
" title="2.2.9. The qualifier @Named at injection points">

### Conclusion: Strategic Choices for Bean Naming and Defaults

In conclusion, the potential pitfalls associated with `@Named` underscore the need for careful
consideration when using this annotation in **CDI**. Ambiguity and unintended defaults can arise
when
relying on implicit naming, especially in the presence of multiple implementations. Developers are
encouraged to explore alternatives such as `@Identifier` from **SmallRye Common** for a more
controlled
and explicit approach to bean identification. Embracing explicit qualification, custom qualifiers,
and alternative approaches ensures a smoother and more controlled CDI experience, leading to robust
and maintainable Java.