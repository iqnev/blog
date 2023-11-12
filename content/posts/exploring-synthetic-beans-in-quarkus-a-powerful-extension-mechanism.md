+++
title = 'Exploring Synthetic Beans in Quarkus: A Powerful Extension Mechanism'
date = 2023-11-11T11:04:18+03:00

tags = ['Development', 'Quarkus', 'CDI', 'Extension']

categories = ['quarkus']

noComment = false
+++

In the world of Quarkus, the realm of dependency injection is rich and versatile, offering
developers a multitude of tools to manage and control beans. One such tool is the concept of
synthetic beans. Synthetic beans are a powerful extension mechanism that allows you to register
beans whose attributes are not derived from a Java class, method, or field. Instead, all the
attributes of a synthetic bean are defined by an extension.

In this article, we'll take a deep dive into the world of synthetic beans in Quarkus. We'll explore
the need for synthetic beans, their practical applications, and how to create and use them in your
Quarkus applications.

## Understanding Synthetic Beans

In Quarkus, beans are the building blocks of your application, managed by the Contexts and
Dependency Injection (CDI) framework. Typically, CDI beans are Java classes that are annotated with
various CDI annotations such as `@ApplicationScoped`, `@RequestScoped`, or `@Inject`. These
annotations
allow CDI to automatically manage the lifecycle and injection of beans.

However, there are situations where you may need to register a bean that doesn't neatly fit into the
traditional CDI model. This is where synthetic beans come into play. Synthetic beans are created by
extensions and have their attributes entirely defined by these extensions. In the world of regular
CDI, you would achieve this using the `AfterBeanDiscovery.addBean()`
and `SyntheticComponents.addBean()`
methods. In Quarkus, this is accomplished using `SyntheticBeanBuildItem`.

## When Do You Need Synthetic Beans?

So, when might you need to use synthetic beans in Quarkus? Synthetic beans are a powerful tool when:

1. __Integrating Third-Party Libraries:__ You're working with a third-party library that doesn't
   have CDI annotations but needs to be integrated into your CDI-based application. Synthetic beans
   allow you to bridge this gap.

2. __Dynamic Bean Registration:__ You need to register beans dynamically at runtime, depending on
   configuration or other factors. Synthetic beans give you the flexibility to create and register
   beans on-the-fly.

3. __Customized Bean Management:__ You require fine-grained control over the scope and behavior of a
   bean that can't be achieved with standard CDI annotations.

4. __Implementing Specialized Beans:__ You want to create specialized beans with unique attributes
   that don't correspond to traditional Java classes or methods.

5. __Mocking Dependencies for Testing:__ Synthetic beans provide a useful way to mock out
   dependencies and inject mock implementations for testing purposes.

## SynthesisFinishedBuildItem

The `SynthesisFinishedBuildItem` is used to indicate that the CDI bean discovery and registration
process has completed.
This allows extensions to know when it is safe to interact with the beans that have been registered.

For example:

```java
@BuildStep  
void onSynthesisFinished(SynthesisFinishedBuildItem synthesisFinished){
    // CDI bean registration is complete, can now safely interact with beans
    }
```

## SyntheticBeansRuntimeInitBuildItem

The `SyntheticBeansRuntimeInitBuildItem` is used to register a callback that will be invoked at
runtime after all synthetic beans have been initialized.
This is useful if you need to perform additional initialization logic involving synthetic beans.

For example:

```java
@BuildStep
SyntheticBeansRuntimeInitBuildItem initSyntheticBeans(){

    return new SyntheticBeansRuntimeInitBuildItem(ids->{
    // Perform logic with initialized synthetic beans
    });

    }
```

The callback passed to `SyntheticBeansRuntimeInitBuildItem` will receive a `Set<Integer>` containing
the IDs of all initialized synthetic beans.

So in summary, `SynthesisFinishedBuildItem` indicates bean discovery is done,
while `SyntheticBeansRuntimeInitBuildItem` allows initializing logic depending on synthetic beans.

## Creating Synthetic Beans with SyntheticBeanBuildItem

In Quarkus, creating synthetic beans is a straightforward process, thanks to
the `SyntheticBeanBuildItem` class.
Let's walk through the steps to create and use a synthetic bean:

1. __Create the Synthetic Bean Class:__ Start by defining the synthetic bean class. This class will
   be the foundation for your synthetic bean.

```java
package com.iqnev;

public class MySyntheticBean {

  // Define the behavior and attributes of your synthetic bean
  public void printMessage() {
    System.out.println("Hello from synthetic bean!");
  }
}
```

2. __Create a Quarkus Extension:__ You'll need to create a Quarkus extension to register your synthetic
   bean.
   This extension class will use `SyntheticBeanBuildItem` to configure your bean.

### Bytecode Generation Approach

```java
package com.iqnev;

import io.quarkus.arc.deployment.SyntheticBeanBuildItem;

public class MySyntheticBeanExtension {

  @BuildStep
  SyntheticBeanBuildItem syntheticBean() {
    return SyntheticBeanBuildItem
        .configure(MySyntheticBean.class)
        .scope(ApplicationScoped.class)
        .creator(mc -> {
          mc.returnValue(new MySyntheticBean());
        })
        .done();
  }
}
```

The `.creator()` method on `SyntheticBeanBuildItem` is used to generate the bytecode that will
create instances of the synthetic bean at runtime.

The argument passed to `.creator()` is a `Consumer<MethodCreator>` which allows generating Java
bytecode inside a method.

In this example:

1. `mc` is the `MethodCreator` instance
2. `mc.returnValue(new MySyntheticBean())` generates the bytecode to create a new instance
   of `MySyntheticBean` and return it from the method.

So essentially, we are telling Quarkus to generate a method that looks something like:

```java
MySyntheticBean createSyntheticBean(){
    return new MySyntheticBean();
    }
```

This generated method will then be called to instantiate the `MySyntheticBean` when it needs to be
injected or used.

The reason bytecode generation is used is that synthetic beans do not correspond to real Java
classes/methods, so we have to explicitly generate a method to instantiate them

The output of `SyntheticBeanBuildItem` is bytecode recorded at build time. This limits how instances
are created at runtime. Common options are:

1. Generate bytecode directly via `.creator()`
2. Use a `BeanCreator` subclass
3. Produce instance via `@Recorder` method

### Recorder Approach

The `@Record` and `.runtimeValue()` approaches are alternate ways of providing instances for
synthetic beans in Quarkus.

This allows you to instantiate the synthetic bean via a recorder class method annotated
with `@Record(STATIC_INIT)`.
For example:

```java

@Recorder
public class MyRecorder {

  @Record(STATIC_INIT)
  public MySyntheticBean createBean() {
    return new MySyntheticBean();
  }

}

  @BuildStep
  SyntheticBeanBuildItem syntheticBean(MyRecorder recorder) {
    return SyntheticBeanBuildItem
        .configure(MySyntheticBean.class)
        .runtimeValue(recorder.createBean());
  }
```

Here the `.runtimeValue()` references the recorder method to instantiate the bean.
__.runtimeValue__

This allows passing a `RuntimeValue` directly to provide the synthetic bean instance.

For example:

```java
@BuildStep 
SyntheticBeanBuildItem syntheticBean(){

    RuntimeValue<MySyntheticBean> bean= //...

    return SyntheticBeanBuildItem
    .configure(MySyntheticBean.class)
    .runtimeValue(bean);

    }
```

The `RuntimeValue` could come from a recorder, supplier, proxy etc.

So in summary:

* `@Record` is one approach to generate the `RuntimeValue`
* `.runtimeValue()` sets the `RuntimeValue` on the `SyntheticBeanBuildItem`

They both achieve the same goal of providing a runtime instance, just in slightly different ways.

When it comes to providing runtime instances for synthetic beans in Quarkus, I would consider using
recorders (via `@Record`) to be a more advanced approach compared to directly generating bytecode
with
`.creator()` or supplying simple RuntimeValues.

Here are some reasons why using recorders can be more advanced:

* __More encapsulation -__ The logic to instantiate beans is contained in a separate recorder class
  rather than directly in build steps. This keeps build steps lean.
* __Reuse -__ Recorder methods can be reused across multiple synthetic beans rather than rewriting
  creator logic.
* __Runtime data -__ Recorder methods execute at runtime so they can leverage runtime resources,
  configs, services etc. to construct beans.
* __Dependency injection -__ Recorder methods can inject other services.
* __Life cycle control -__ Recorder methods annotated with `@Record(STATIC_INIT)`
  or `@Record(RUNTIME_INIT)` give more control over bean instantiation life cycle.
* __Managed beans -__ Beans instantiated inside recorders can themselves be CDI managed beans.

So in summary, recorder methods provide more encapsulation, flexibility and access to runtime data
and services for instantiating synthetic beans. They allow for more advanced bean production logic
compared to direct bytecode generation.

However, direct bytecode generation with `.creator()` can still be useful for simple cases where
recorders may be overkill. But as synthetic bean needs grow, recorders are a more powerful and
advanced approach.

*__[IMPORTANT]__*
It is possible to configure a synthetic bean in Quarkus to be initialized during the `RUNTIME_INIT`
phase instead of the default `STATIC_INIT` phase.

Here is an example:

```java
@BuildStep
@Record(RUNTIME_INIT)
SyntheticBeanBuildItem lazyBean(BeanRecorder recorder){

    return SyntheticBeanBuildItem
    .configure(MyLazyBean.class)
    .setRuntimeInit() // initialize during RUNTIME_INIT
    .runtimeValue(recorder.createLazyBean());

    }
```

The key points are:

* Use `setRuntimeInit()` on the `SyntheticBeanBuildItem` to mark it for `RUNTIME_INIT`
* The recorder method must be annotated with `@Record(RUNTIME_INIT)`
* The runtime init synthetic beans cannot be accessed during `STATIC_INIT`

So in summary, synthetic beans can be initialized lazily during `RUNTIME_INIT` for cases where
eager `STATIC_INIT` instantiation is not needed. This allows optimizing startup time.

3. __Use the Synthetic Bean:__ Now that your synthetic bean is registered, you can inject and use it
   in your application.

```java
package com.iqnev;

import javax.inject.Inject;

public class MyBeanUser {

  @Inject
  MySyntheticBean mySyntheticBean;

  public void useSyntheticBean() {
    // Use the synthetic bean in your code
    mySyntheticBean.printMessage();
  }
}
```

4. Running Your Application: Build and run your Quarkus application as usual, and the synthetic bean
   will be available for injection and use.

## Conclusion

Synthetic beans in Quarkus provide a powerful mechanism for integrating external libraries,
dynamically registering beans, and customizing bean behavior in your CDI-based applications. These
beans, whose attributes are defined by extensions rather than Java classes, offer flexibility and
versatility in managing dependencies.

As we've explored in this article, creating and using synthetic beans in Quarkus is a
straightforward process. By leveraging `SyntheticBeanBuildItem` and Quarkus extensions, you can
seamlessly bridge the gap between traditional CDI and more specialized or dynamic bean registration
requirements.

In the ever-evolving landscape of Java frameworks, Quarkus continues to stand out by offering
innovative solutions like synthetic beans, making it a compelling choice for modern, efficient, and
flexible application development. Embrace the power of synthetic beans in Quarkus, and take your
dependency injection to the next level!

