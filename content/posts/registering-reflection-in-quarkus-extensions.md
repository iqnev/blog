+++
title = 'Registering Reflection in Quarkus Extensions'
date = 2023-11-26T14:04:18+03:00

tags = ['Development', 'Quarkus', 'Extension', 'GraalVM']

categories = ['quarkus']

noComment = false
+++

Quarkus utilizes ahead-of-time **(AOT)** compilation to build blazing fast native executables. However,
**AOT** works through closed-world analysis which eliminates unused code paths. This can break
functionality relying on runtime reflection like dependency injection, bytecode manipulation, and
integration with certain libraries.

## Registering for Reflection
When building a native executable, GraalVM operates under a closed-world assumption, analyzing the
call tree and eliminating unused classes, methods, and fields. To include elements requiring
reflective access, explicit registration becomes crucial.

Using the `@RegisterForReflection` Annotation
The simplest way to register a class for reflection is through the `@RegisterForReflection`
annotation:

```java
@RegisterForReflection
public class MyClass {
}
```

For classes in third-party JARs, an empty class can host the `@RegisterForReflection` annotation:

```java
@RegisterForReflection(targets={ DemoReflection1.class, DemoReflection2.class})
public class MyReflectionConfiguration {
}
```

Note that `DemoReflection1` and `DemoReflection2` will be registered for
reflection, but not `MyReflectionConfiguration`.

Using a Configuration File
Configuration files can also be used to register classes for reflection. For instance, to register
all methods of `com.demo.MyClass`, create `reflection-config.json`:

```json
[
{
"name" : "com.demo.MyClass",
"allDeclaredConstructors" : true,
"allPublicConstructors" : true,
"allDeclaredMethods" : true,
"allPublicMethods" : true,
"allDeclaredFields" : true,
"allPublicFields" : true
}
]
```


Make the configuration file known to the native-image executable by adding the following to
`application.properties`:

```properties
quarkus.native.additional-build-args=-H:ReflectionConfigurationFiles=reflection-config.json
```


## Quarkus Extension Support for Native Mode

To enable native mode support for a custom extension, Quarkus simplifies the registration of
reflection through `ReflectiveClassBuildItem`.
This class is used in the build process to specify classes requiring reflective access.

## Understanding ReflectiveClassBuildItem:
`ReflectiveClassBuildItem` is a Quarkus-specific class utilized in the extension development process.
It plays a crucial role in indicating which classes should be made available for reflective access
at runtime. This is especially relevant when certain operations, such as dependency injection or
bytecode manipulation, require runtime reflection.

## Usage in Quarkus Extensions:
When creating a Quarkus extension, you can seamlessly integrate the registration of reflective
classes using `ReflectiveClassBuildItem`.
The `@BuildStep` annotation signifies a build step, a fundamental concept in Quarkus extension
development.

```java
public class MyClass {

    @BuildStep
    ReflectiveClassBuildItem reflection() {
        // Since reflection is needed only for the constructor, `false` is specified for both methods and fields arguments.
        return new ReflectiveClassBuildItem(false, false, "com.demo.DemoClass");
    }

}
```


In this snippet, `MyClass` is a placeholder for the actual extension class you are
developing. The `reflection()` method, annotated with `@BuildStep`, creates an instance of
`ReflectiveClassBuildItem`, indicating that the class `com.demo.DemoClass` requires reflective access.
The false arguments for methods and fields indicate that reflective access is needed only for the
constructor.

I showcase a Quarkus extension that leverages the `ReflectiveClassBuildItem to` dynamically register
classes for reflection.
The extension focuses on identifying classes implementing a specific interface (Conversion in this
case) and also explicitly registers some standard Java classes for reflective access.

```java
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.CombinedIndexBuildItem;
import io.quarkus.deployment.builditem.nativeimage.ReflectiveClassBuildItem;
import org.jboss.jandex.ClassInfo;

import java.text.DecimalFormat;
import java.text.DecimalFormatSymbols;
import java.text.SimpleDateFormat;

public class ReflectionExtension {

    // Interface to identify classes for reflection
    private static final DotName CUSTOM_FEATURE_INTERFACE = DotName.createSimple(CustomFeature.class.getName());

    @BuildStep
    void registerForReflection(CombinedIndexBuildItem combinedIndex,
                               BuildProducer<ReflectiveClassBuildItem> reflectiveClasses) {

        for (ClassInfo implClassInfo : combinedIndex.getIndex().getAllKnownImplementors(CUSTOM_FEATURE_INTERFACE)) {
            String combinedIndexName = implClassInfo.name().toString();
            log.debugf("CustomFeature class implementation '[%s]' registered for reflection", combinedIndexName);

            reflectiveClasses.produce(new ReflectiveClassBuildItem(true, true, combinedIndexName));
        }

    }

}
```

**Explanation:**

1. **CombinedIndexBuildItem:** This build item provides access to the combined index of all classes in
   the application. In this example, it is used to retrieve all known implementors of the Conversion
   interface.

2. **Iterating Over Implementors:** The extension iterates over all classes implementing the Conversion
   interface and registers them for reflection using `ReflectiveClassBuildItem`.

3. **DotName** is a class representing a dotted name, which is essentially a fully qualified class name
   in a format where package names and class names are separated by dots. The DotName class is part
   of the **Jandex library**, which is a tool used by Quarkus for indexing and querying Java classes.
   `DotName` is used to represent and work with fully qualified class names in the **Jandex indexing**
   system. It's a lightweight and efficient way to refer to classes within the **Jandex index**.

**Considerations:**
While `ReflectiveClassBuildItem` provides a mechanism to address reflective access requirements, it's
crucial to use it judiciously. Excessive reliance on reflective access can undermine the performance
benefits of Quarkus' AOT compilation approach. Therefore, it's recommended to leverage this tool
sparingly and explore alternative strategies whenever possible.

In summary, understanding and effectively using ReflectiveClassBuildItem is key to optimizing
Quarkus extensions for native mode. By selectively indicating classes that necessitate reflective
access, developers can strike a balance between the advantages of AOT compilation and the
unavoidable realities of certain runtime operations.


