---
title: 'Apache Camel Pulsar Function, Part 1'
date: 2023-06-15
draft: false
tags:
- pulsar
- camel
- lowcode
- functions
---

I recently came across an interesting [article from DataStax](https://thenewstack.io/simplified-data-pipelines-with-pulsar-transformation-functions) about simplified, low-code friendly data piepelines with [Pulsar Function](https://pulsar.apache.org/docs/functions-overview) and since I always wanted to learn a little bit more about [Apache Pulsar](https://pulsar.apache.org) and I've been working on something similar, I've [started exploring](https://github.com/lburgazzoli/camel-pulsar-function) how a Pulsar Function based on [Apache Camel](https://camel.apache.org) would look like.

# Background

To get started, let's have a basic understanding of Apache Pulsar and Apache Camel:

- [Apache Pulsar](https://pulsar.apache.org) is an open-source, distributed messaging and streaming platform built for the cloud. 
- [Apache Pulsar Functions](https://pulsar.apache.org/docs/3.0.x/functions-overview/) is lightweight serverless computing framework enabling developers to process and manipulate data in real-time. The framework takes care of the underlying details of sending and receiving messages. You only need to focus on the business logic
- [Apache Camel](https://camel.apache.org) is an open-source integration framework that provides a wide range of connectors and patterns for integrating various systems and applications.

# Goals

Provide a low-code, operational friendly way to deploy data transformation and routing pipelines on top of [Apache Pulsar](https://pulsar.apache.org) leveraging [Apache Camel DSL](https://camel.apache.org/manual/dsl.html)

# Pulsar Function Implementation

Pulsar functions can be developed in [Java, Python, or Go](https://pulsar.apache.org/docs/3.0.x/functions-develop-api/) and since [Apache Camel](https://camel.apache.org) is a Java framework, we can leveraging the Java SDK to embed a Camel Routing Engine in the Function Runtime:

![camel-embedded](/images/pulsar-function-camel-embedded.png)

In addition to the standard Java SDK, there is also an [Extended SDK for Java](https://pulsar.apache.org/docs/3.0.x/functions-develop-api/#use-extended-sdk-for-java) which provides two additional interfaces to initialize and release external resources:
- With the `initialize` interface, you can initialize external resources which only need one-time initialization when the function instance starts.
- With the `close` interface, you can close the referenced external resources when the function instance closes.

In order to embed the Apache Camel in the function runtime, an instance of a [CamelContext](https://camel.apache.org/manual/camelcontext.html), must be created and since it is an heavyweight object, we can tie its lifecycle to the `initialize` and `close` so a CamelContext is created and started only once when the function instance starts and then it last for the entire lifecycle of the function.

At an high level, the [implementation](https://github.com/lburgazzoli/camel-pulsar-function) does the following tasks:
1. Initialize and start a long running CamelContext instance when the function instance starts;
2. Load the user defined processing steps defined using the [Apache Camel YAML DSL](https://camel.apache.org/components/3.20.x/others/yaml-dsl.html);
3. Wire the function input to the Camel's Route and publish back the result to Pulsar depending on the configuration and routing decision;
4. Close the CamelContext instance when the function instance closes. 

# Pulsar Function Configuration

The `CamelFunction` reads its configuration as `YAML` from the Function `userConfig` `steps` parameter.
As an example, a [function configuration file](https://pulsar.apache.org/docs/3.0.x/functions-cli/) would look like:

```yaml
tenant: "public"
namespace: "default"
name: "content-based-routing"
inputs:
  - "persistent://public/default/input-1"
output: "persistent://public/default/output-1"
jar: "build/libs/camel-pulsar-function-${version}-all.jar"
className: "com.github.lburgazzoli.camel.pulsar.CamelFunction"
logTopic: "persistent://public/default/logging-function-logs"
userConfig:
  steps: |
    - setHeader:
        name: "source"
        jq: '.source'
    - choice:
        when:
        - jq: '.source == "sensor-1"'
          steps:
          - setProperty:
              name: 'pulsar.apache.org/function.output'
              constant: 'far'
          - setBody:
              jq:
                expression: '.data'
                resultType: 'java.lang.String'
        - jq: '.source == "sensor-2"'
          steps:
          - setProperty:
              name: 'pulsar.apache.org/function.output'
              constant: 'near'
          - setBody:
              jq:
                expression: '.data'
                resultType: 'java.lang.String'
```

If you are familiar with the [Apache Camel YAML DSL](https://camel.apache.org/components/3.20.x/others/yaml-dsl.html), you have probably noticed that the processing pipeline does not begin with `from` or `route` as you would probably expect and this is because the Camel Pulsar Function automatically create all the boilerplate that are required to wire the Camel's routing engine to the Pulsar Function runtime. Beside this small difference, all the [Apache Camel EIPs](https://camel.apache.org/components/3.20.x/eips/enterprise-integration-patterns.html) are available.

# Access to contextual data

Some attributes of the function and record being processed are mapped to Camel's Exchange Properties:

| Pulsar                   | Exchange Property Name                   |
|:-------------------------|:-----------------------------------------|
| Function ID              | pulsar.apache.org/function.id            |
| Configured Output Topic  | pulsar.apache.org/function.output        |
| Record Topic             | pulsar.apache.org/record.topic           |
| Record Schema            | pulsar.apache.org/record.schema          |
| Record Key               | pulsar.apache.org/record.key             |
| Record Partition ID      | pulsar.apache.org/record.partition.id    |
| Record Partition Index   | pulsar.apache.org/record.partition.index |

# Access to Record data

Any Function's Record property is mapped to an Camel's Message Header

# Routing

By default, the result of the processing pipeline is sent to the output topic defined in the function configuration, however it is possible to pick a different topic to apply a [Content-Based Routing pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ContentBasedRouter.html) as show in the example, by setting the exchange property `pulsar.apache.org/function.output`


# Building and Deploying the Pulsar Function

1. Clone the repository https://github.com/lburgazzoli/camel-pulsar-function
2. Build the project by executing  `./gradlew clean shadowJar`
3. Deploy the generated function artifact (`build/libs/camel-pulsar-function-0.1.0-SNAPSHOT-all.jar`) by following the [tutorial](https://pulsar.apache.org/docs/3.0.x/functions-deploy/)

# Conclusion

Event this is nothing more than a POC which requires more work to be production ready, I think that by combining Pulsar Function and Apache Camel's integration capabilities, developers can build robust and efficient data pipelines with ease. 
The seamless integration, versatile transformation capabilities, flexible routing, and extensibility offered by this implementation make it an excellent choice for simplifying data pipelines within the Pulsar ecosystem.

To learn more about this implementation and explore its code, visit the [GitHub repository](https://github.com/lburgazzoli/camel-pulsar-function)

# TODO

- [ ] Automatic data type conversion
- [ ] NAtive Support for Apache Pulsar in [Apache Camel K](https://camel.apache.org/camel-k/1.12.x/index.html)
- [ ] Experiment with [Pulsar Function Mesh](https://functionmesh.io/)
