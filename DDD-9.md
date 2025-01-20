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

Reuse of ConnectorMetadataProvider, ConnectorMetadata, ConnectorDescriptor, Field

Pro -> Field has also the concept of deprecation
Cons-> Custom components will not be automatically supported but users need to provide the descriptor.

or 

use the ConfigDef and ConfigKey.

Pro -> Currently all the Kafka Connect components exposes the ConfigDef, there should just be homogenized putting
that method in a common interface, so they can be easily discovered. Also, custom components will benefit from it.

or a mix of both? Maybe using a more generic interface for discover components that provide configuration definition.
