# DDD-9: Debezium component descriptors / registry

## Motivation
Debezium platform project has been started with the idea to provide a centralized and easy way to manage data pipelines on Kubernetes with Debezium Server.
In that platform, users can configure the different components: sources, sinks and transformations, to build their data pipelines. 
For those are familiar with Debezium, these components are highly configurable through a set of configuration properties. 
These properties are declared by the components itself and usually a property defines some basic infos:

* name
* description
* type
* default value
* validator

These property are generally documented on the component docs that is enough if you configure it in the standard way (Kafka Connect, Debezium Server, Debezium engine) but is not the case for the Debezium Platform.

The UI of the Debezium Platform (stage) needs a way to get all the possible configuration properties declared by a component to easily permit to the users to configure them.

This document will present how this can be achieved. 

## Goals

The goal is to define a descriptor for Connectors, Transformations and Sinks (and storage?) and how this will be created and exposed to the stage. 

## Descriptors format

Descriptor format should be expressive and human writable/readable. The first thought goes to JSON and YAML.
Let's explore how the different format would be for describing connector properties. 

```json
{
  "name": "PostgreSQL Connector",
  "type": "source-connector",
  "version": "1.0.0",
  "metadata": {
    "description": "Captures changes from a PostgreSQL database",
    "tags": ["database", "postgresql", "cdc"]
  },
  "properties": [
    {
      "name": "topic.prefix",
      "type": "string",
      "required": true,
      "display": {
        "label": "Topic prefix",
        "description": "Topic prefix that identifies and provides a namespace for the particular database server/cluster is capturing changes. The topic prefix should be unique across all other connectors, since it is used as a prefix for all Kafka topic names that receive events emitted by this connector. Only alphanumeric characters, hyphens, dots and underscores must be accepted.",
        "group": "Connection",
        "groupOrder": 0,
        "width": "medium",
        "importance": "high"
      },
      "validation": [
        {
          "type": "regex",
          "pattern": "^[a-zA-Z0-9._-]+$",
          "message": "Only alphanumeric characters, hyphens, dots and underscores are allowed"
        }
      ]
    },
    {
      "name": "table.include.list",
      "type": "list",
      "display": {
        "label": "Include Tables",
        "description": "The tables for which changes are to be captured",
        "group": "Filters",
        "groupOrder": 2,
        "width": "long",
        "importance": "high"
      },
      "validation": [
        {
          "type": "regex-list",
          "message": "Each item must be a valid regular expression"
        }
      ]
    }
  ],
  "groups": [
    {
      "name": "Connection",
      "order": 0,
      "description": "Connection configuration for the PostgreSQL database"
    },
    {
      "name": "Filters",
      "order": 2,
      "description": "Filtering options for tables and changes"
    }
  ]
}
```

the same in YAML

```yaml
name: PostgreSQL Connector
type: source-connector
version: 1.0.0
metadata:
  description: Captures changes from a PostgreSQL database
  tags: 
    - database
    - postgresql
    - cdc

groups:
  - name: Connection
    order: 0
    description: Connection configuration for the PostgreSQL database
  - name: Filters
    order: 2
    description: Filtering options for tables and changes

properties:
  - name: topic.prefix
    type: string
    required: true
    display:
      label: Topic prefix
      description: >
        Topic prefix that identifies and provides a namespace for the particular database 
        server/cluster is capturing changes. The topic prefix should be unique across 
        all other connectors, since it is used as a prefix for all Kafka topic names 
        that receive events emitted by this connector. Only alphanumeric characters, 
        hyphens, dots and underscores must be accepted.
      group: Connection
      groupOrder: 0
      width: medium
      importance: high
    validation:
      - type: regex
        pattern: ^[a-zA-Z0-9._-]+$
        message: Only alphanumeric characters, hyphens, dots and underscores are allowed

  - name: table.include.list
    type: list
    display:
      label: Include Tables
      description: The tables for which changes are to be captured
      group: Filters
      groupOrder: 2
      width: long
      importance: high
    validation:
      - type: regex-list
        message: Each item must be a valid regular expression
```

a custom DSL

```text
connector "PostgreSQL Connector" {
  type: source-connector
  version: "1.0.0"
  tags: [database, postgresql, cdc]
  description: "Captures changes from a PostgreSQL database"
}

group Connection {
  order: 0
  description: "Connection configuration for the PostgreSQL database"

  property "topic.prefix" {
    type: string
    required: true
    label: "Topic prefix"
    width: medium
    importance: high
    description: """
      Topic prefix that identifies and provides a namespace for the particular database 
      server/cluster is capturing changes. The topic prefix should be unique across 
      all other connectors, since it is used as a prefix for all Kafka topic names 
      that receive events emitted by this connector. Only alphanumeric characters, 
      hyphens, dots and underscores must be accepted.
    """
    validate {
      regex("^[a-zA-Z0-9._-]+$", "Only alphanumeric characters, hyphens, dots and underscores are allowed")
    }
  }
}

group Filters {
  order: 2
  description: "Filtering options for tables and changes"

  property "table.include.list" {
    type: list
    label: "Include Tables"
    width: long
    importance: high
    description: "The tables for which changes are to be captured"
    validate {
      regex-list("Each item must be a valid regular expression")
    }
  }
}
```

We exclude more complex format like [OAS](https://swagger.io/specification/), although it is used by `debezium-schema-generator`, since it is too complicated for our scope. 

## Generate descriptors

### Debezium schema generator
The old UI used a json descriptor built by the `debezium-schema-generator`. The `SchemaGenerator` class based its generation on the following interfaces

```java
public interface ConnectorMetadataProvider {

    ConnectorMetadata getConnectorMetadata();
}

public interface ConnectorMetadata {

    ConnectorDescriptor getConnectorDescriptor();

    Field.Set getConnectorFields();
}
```
Each connector needs to implement these two interfaces to provide the information required for the descriptor generation. 

The `ConnectorDescriptor` class contains some basic information like the `className`, `version`, `displayName` and `id`.
The `Field` class contains the information used to describe a configuration property. Below there is an extract of the properties of this class.

```java
    private final String name;
    private final String displayName;
    private final String desc;
    private final Supplier<Object> defaultValueGenerator;
    private final Validator validator;
    private final Width width;
    private final Type type;
    private final Importance importance;
    private final List<String> dependents;
    private final Recommender recommender;
    private final java.util.Set<?> allowedValues;
    private final GroupEntry group;
    private final boolean isRequired;
    private final java.util.Set<String> deprecatedAliases;
```

| Pro                           | Cons                                                     |
|-------------------------------|----------------------------------------------------------|
| Alias and deprecation support | Requires change to the components code                   |
| Decoupled from Kafka Connect  | Loose control on custom connectors, transformations, etc |


### Kafka Connect data

Every Kafka Connect components exposes the `ConfigDef` object that contains the definition of configuration property that the component declare. 
A configuration is mapped in the `ConfigKey` class. 

```java
public static class ConfigKey {
    public final String name;
    public final Type type;
    public final String documentation;
    public final Object defaultValue;
    public final Validator validator;
    public final Importance importance;
    public final String group;
    public final int orderInGroup;
    public final Width width;
    public final String displayName;
    public final List<String> dependents;
    public final Recommender recommender;
    public final boolean internalConfig;
    public final String alternativeString;

    //...
}
```

| Pro                              | Cons                                        |
|----------------------------------|---------------------------------------------|
| Discoverability of components    | Coupled with Kafka Connect                  |
| No change to the components code | Missing support for deprecation and aliases |

The `Discoverability of components` requires an in-depth analysis. 
The `ConfigDef config()` method is declared in different interfaces/classes, and so the discoverability means to look for all of that.  

```java
public abstract class Connector implements Versioned {
    
    /**
     * Define the configuration for the connector.
     * @return The ConfigDef for this connector; may not be null.
     */
    public abstract ConfigDef config();
}

public interface Converter {

    /**
     * Configuration specification for this converter.
     * @return the configuration specification; may not be null
     */
    default ConfigDef config() {
        return new ConfigDef();
    }
}

public interface Transformation<R extends ConnectRecord<R>> extends Configurable, Closeable {

    /** Configuration specification for this transformation. */
    ConfigDef config();
}

public interface Predicate<R extends ConnectRecord<R>> extends Configurable, AutoCloseable {

    /**
     * Configuration specification for this predicate.
     *
     * @return the configuration definition for this predicate; never null
     */
    ConfigDef config();
}
```

We could propose a KIP to move this method in a dedicated interface

```java
/**
 * ConfigSpecifier has the sole responsibility of defining what configuration a component accepts
 */
public interface ConfigSpecifier {
   
    /** Configuration specification. */
    ConfigDef config();
}
```

to easily discover all classes that implements this interface. 

### Considerations 

The approach to use the `Kafka Connect` classes, even if it is coupled to it, offers more flexibility in supporting components outside Debezium.
The only real cons is the missing of a deprecation and aliases support. Can we think to contribute and add the support to `ConfigKey`?

Then the `debezium-schema-generator` could be modified to support the new format and how retrieve the components.

## Storing descriptor

Since the main user of that descriptor, as of now, will be the Debezium Platform, them cannot be got from a running debezium server instance,
since the information contained in the description needs to be used during the creation of the Debezium Platform components (Source, Destination and Transformation).

This requires to store/publish the generated descriptor somewhere. 


