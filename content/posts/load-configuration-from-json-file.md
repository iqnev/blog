+++
title = 'Creating Custom Configuration in Quarkus Loaded from JSON File'
date = 2023-10-14T21:00:18+03:00

tags = ['Development', 'Quarkus', 'MicroProfile', 'JSON']

categories = ['quarkus']

noComment = false
+++

### Introduction

Quarkus, a framework for building lightweight, fast, and efficient Java applications, offers
developers the flexibility to create custom configurations loaded from JSON files. These custom
configurations can be seamlessly integrated into your Quarkus application, enhancing its
configurability and adaptability. To achieve this, Quarkus utilizes Eclipse MicroProfile Config (
MP-Config), with the SmallRye implementation providing the necessary tools. In this article, we'll
delve into the process of crafting custom configurations and loading them from JSON files within
Quarkus, all while exploring the mechanics of SmallRye's MP-Config implementation. Additionally,
we'll showcase the creation and registration of a custom ConfigSource and ConfigSourceFactory, which
play a pivotal role in this configuration management approach.

### Understanding MicroProfile Config in Quarkus

Quarkus internally relies on the SmallRye implementation of MP-Config. This implementation allows
developers to incorporate Configuration Sources, which provide configuration data from various
origins. These sources can be files with non-standard formats, or even data retrieved from a central
repository. MP-Config ensures a deterministic ordering of configuration sources based on their
ordinal values when multiple sources contain the same configuration key.

SmallRye's implementation of MP-Config facilitates the creation of new ConfigSources and
ConfigSourceFactories. The ConfigSourceFactory has knowledge of all previously defined sources,
enabling developers to read those values and pass them to a newly created ConfigSource. The
registration of these custom sources and factories is accomplished through the Java ServiceLoader
interface, with a specific file called `io.smallrye.config.ConfigSourceFactory` being placed in the
`META-INF/services/` directory. This file provides the fully qualified names of the custom sources
and
factories.

### Implementing a custom JSON Configuration

To demonstrate the creation and registration of a custom `ConfigSource` and `ConfigSourceFactory` in
Quarkus, we'll focus on a practical example - an `JsonConfigSource` and `JsonConfigSourceFactory`
pair.
This custom configuration source allows you to read external JSON configuration files and integrate
them into your Quarkus application.

The `JsonConfigSource` class is responsible for reading and providing configuration properties from
a JSON file. It implements the `ConfigSource` interface and overrides several methods to interact
with
the Quarkus configuration system.

Here is an overview of its key functionalities:

* Reading a JSON file and parsing it into a JsonObject.
* Providing configuration properties, including handling default values.
* Specifying the source's ordinal value.
* Assigning a unique name to the source, which will be used for registration.

```java

@Slf4j
public class JsonConfigSource implements ConfigSource {

  private final Map<String, ConfigValue> existingValues;

  private JsonObject root;

  public JsonConfigSource(final Map<String, ConfigValue> exProp) {
    existingValues = exProp;
  }

  public void addJsonConfigurations(final ConfigValue config) {
    final File file = new File(config.getValue());

    if (!file.canRead()) {
      log.warn("Can't read config from " + file.getAbsolutePath() + "");
    } else {
      try (final InputStream fis = new FileInputStream(file);
          final JsonReader reader = Json.createReader(fis)) {

        root = reader.readObject();
      } catch (final IOException ioe) {
        log.warn("Reading the config failed: " + ioe.getMessage());
      }
    }
  }

  @Override
  public Map<String, String> getProperties() {

    final Map<String, String> props = new HashMap<>();
    final Set<Map.Entry<String, ConfigValue>> entries = existingValues.entrySet();
    for (final Map.Entry<String, ConfigValue> entry : entries) {
      String newVal = getValue(entry.getKey());
      if (newVal == null) {
        newVal = entry.getValue().getValue();
      }
      props.put(entry.getKey(), newVal);
    }

    return props;
  }

  @Override
  public Set<String> getPropertyNames() {
    return existingValues.keySet();
  }

  @Override
  public int getOrdinal() {
    return 270;
  }

  @Override
  public String getValue(final String configKey) {

    final JsonValue jsonValue = root.get(configKey);

    if (jsonValue != null) {
      return getStringValue(jsonValue);
    }

    if (existingValues.containsKey(configKey)) {
      return existingValues.get(configKey).getValue();
    } else {
      return null;
    }
  }

  @Override
  public String getName() {
    return "EXTERNAL_JSON";
  }

  private String getStringValue(final JsonValue jsonValue) {
    if (jsonValue != null) {
      final JsonValue.ValueType valueType = jsonValue.getValueType();

      if (valueType == JsonValue.ValueType.STRING) {
        return ((JsonString) jsonValue).getString();
      } else if (valueType == JsonValue.ValueType.NUMBER) {
        // Handle integer and floating-point numbers
        return jsonValue.toString();
      } else if (valueType == JsonValue.ValueType.TRUE || valueType == JsonValue.ValueType.FALSE) {
        // Handle boolean values
        return Boolean.toString(jsonValue.getValueType() == JsonValue.ValueType.TRUE);
      } else if (valueType == JsonValue.ValueType.NULL) {
        // Handle null values
        return null;
      }
    }
    return null;
  }
}
```

The `JsonConfigSourceFactory` class is a custom `ConfigSourceFactory` responsible for creating and
configuring instances of the `JsonConfigSource`. It also defines a unique priority for this factory.

Here is an overview of its key functionalities:

* Retrieving the path of the JSON configuration file from the Quarkus configuration.
* Building an instance of JsonConfigSource.
* Assigning a priority value to the factory.

```java

@Slf4j
public class JsonConfigSourceFactory implements ConfigSourceFactory {

  public static final String CONFIG_JSON_FILE = "config.json.file";

  @Override
  public Iterable<ConfigSource> getConfigSources(final ConfigSourceContext configSourceContext) {
    final ConfigValue value = configSourceContext.getValue(CONFIG_JSON_FILE);

    if (value == null || value.getValue() == null) {
      return Collections.emptyList();
    }

    final Map<String, ConfigValue> exProp = new HashMap<>();
    final Iterator<String> stringIterator = configSourceContext.iterateNames();

    while (stringIterator.hasNext()) {
      final String key = stringIterator.next();
      final ConfigValue cValue = configSourceContext.getValue(key);
      exProp.put(key, cValue);
    }

    final JsonConfigSource configSource = new JsonConfigSource(exProp);
    final List<ConfigValue> configValueList = List.of(value);

    for (final ConfigValue config : configValueList) {
      if (ConfigExists(config)) {
        configSource.addJsonConfigurations(config);
      }
    }

    return Collections.singletonList(configSource);
  }

  @Override
  public OptionalInt getPriority() {
    return OptionalInt.of(270);
  }

  private boolean ConfigExists(final ConfigValue config) {

    if (config == null || config.getValue() == null) {
      log.warn("The given ConfigValue object is null");
      return false;
    } else if (!(Files.exists(Path.of(config.getValue())))) {
      return false;
    }

    return true;
  }
}
```

### Registration via ServiceLoader

Both the `JsonConfigSource` and `JsonConfigSourceFactory` are registered with the Quarkus
application through the Java ServiceLoader mechanism. A file
named `io.smallrye.config.ConfigSourceFactory` is placed in the `META-INF/services/` directory. This
file contains the fully qualified name of the `JsonConfigSourceFactory`, enabling Quarkus to
discover
and use it.

### The JSON file

```json
{
  "simple.service": "pusher",
  "simple.source": "source",
  "simple.destination": "destination"
}
```

### Define Your Configuration Interface

```java
@ConfigMapping(prefix = "simple")
public interface SimpleConfig {
  @WithName("source")
  String source();

  @WithName("service")
  String service();

  @WithName("destination")
  String destination();
}
```

## Conclusion

Eclipse MicroProfile Config, in conjunction with the SmallRye implementation, empowers Quarkus
developers to manage their application's configuration efficiently. The ability to create and
register custom `ConfigSources` and `ConfigSourceFactories`, as demonstrated with
the `JsonConfigSource`
and `JsonConfigSourceFactory`, extends the flexibility and utility of the framework. By following
these guidelines, you can seamlessly integrate external configuration data, such as JSON files, into
your Quarkus application, enhancing its configurability and adaptability.

By understanding these concepts and leveraging the power of MicroProfile Config, you can further
optimize your Quarkus application's configuration management and streamline the development process.
