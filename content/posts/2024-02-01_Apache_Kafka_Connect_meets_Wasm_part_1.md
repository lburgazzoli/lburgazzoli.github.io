---
title: 'Apache Kafka Connect meets Wasm - Part 1'
date: 2024-02-01
draft: false
tags:
- wasm
- kafka
- integration
---

I recently wrote a blog post about [adding support for Wasm in Apache Camel](https://lburgazzoli.github.io/posts/2024-01-14_apache_camel_meets_wasm_part_1/) with the goal to support WebAssembly as a means to provide custom processing and transformation logic within Apache Camel. 

In this post I'm going to experiment a little bit about doing the same but for Apache Kafka Connect.

# Background

To get started, let’s have a basic understanding of the components/technologies we are going to mention in this post:

- [Apache Kafka Connect](https://kafka.apache.org/documentation/#connect) is a tool for scalably and reliably streaming data between Apache Kafka and other systems. Among other things, Kafka Connect can be configured with transformations to make lightweight message-at-a-time modifications and event routing.
- [WebAssembly (Wasm)](https://webassembly.org) is a low-level bytecode format designed as a portable target for the compilation of high-level languages like C, C++, and Rust, enabling deployment on the web for client and server applications.
- [Chicory](https://github.com/dylibso/chicory) is a JVM native WebAssembly runtime. It allows you to run WebAssembly programs with zero native dependencies or JNI. Chicory can run Wasm anywhere that the JVM can go.

# Goals

Our primary objective is to support **Wasm** as a means to provide custom transformations for **Apache Kafka Connect**.

# Kafka Connect Transformations

Transformations, also referred to as single message transformations (SMT), are entities always associated with a **Connector** that allow to transform messages, one at a time, as they flow through **Kafka Connect**:

![kafka-connect-pipline](/images/kafka-connect-pipeline.svg)

- for source connectors, transformations are invoked after the connector and before the converter.
- for sink connectors, transformations are invoked after the converter and before the connector,

# Implementation

To implement a custom **Kafka Connect** transformation, we must implement the following Java interface

```java
package org.apache.kafka.connect.transforms;

import org.apache.kafka.common.Configurable;
import org.apache.kafka.common.config.ConfigDef;
import org.apache.kafka.connect.connector.ConnectRecord;

import java.io.Closeable;

public interface Transformation<R extends ConnectRecord<R>> extends Configurable, Closeable {

    /**
     * Apply transformation to the {@code record} and return another record object (which may be {@code record} itself)
     * or {@code null}, corresponding to a map or filter operation respectively.
     *
     * @param record the record to be transformed; may not be null
     * @return the transformed record; may be null to indicate that the record should be dropped
     */
    R apply(R record);
}
```

## Memory Management and Data Exchange

In _Wasm_, sharing objects between the host, in this case the _JVM_, and the _Wasm_ module is deliberately restricted and as of today, it requires a number of steps, as example a tipical approach is to:

1. From the _host_, call a function inside the webassembly module that allocates a block of memory and returns its address, then save it
2. From the _host_, write the data that should be exchanged with the _Wasm_ module to the saved address
3. From the _host_, invoke the required function passing both the address where the data is written and its size
4. From the _Wasm_ module, read the data and process it
5. From the _host_, release the memory when done

![wasm-data-sharing](/images/wasm-data-sharing.png)

However, in the context of a **Kafka Connect** transformation, that would require to serialize the entire record and deserialize it back which can be very expensive, so I decided to let the transformation function retrieve only the parts of the record that are of interest by exposing a series of **Host Functions**. 

As the name suggests, a **Host Function** is a function defined in the host program, which can be called by the Wasm module instance.
As an example, a function to retrieve the record's value on the host side would look like:

```java
private Value[] getValueFn(Instance instance, Value... args) {
    final R record = getRecord();
    final byte[] data = recordConverter.fromConnectValue(record);

    int rawDataAddr = alloc.apply(Value.i32(data.length))[0].asInt();

    instance.memory().write(rawDataAddr, data);

    long ptrAndSize = rawDataAddr;
    ptrAndSize = ptrAndSize << 32;
    ptrAndSize = ptrAndSize | data.length;

    return new Value[] {
        Value.i64(ptrAndSize)
    };
}
```

In the guest language, the way to declare such function cahnges from language to language, for rust is a simple as:

```rust
extern "C" {
	fn set_value(ptr: *const u8, len: i32);
}
```

The machinery explained before about sharing data does not change much, except that the _Wasm_ module instance is now the actor that coordinate the data exchange and memory cleanup

![kafka-connect-host-fn-1](/images/kafka-connect-host-fn-1.svg)


## Transformation implementation

As the data exchange pattern relies on writing and readying raw bytes to the _Wasm's_ linear memory as illustrated above, the individual record parts must be serialized/deserialized.
How to do so depends on the message in transit, and we can leverage the existing strategies provided by Kafka Connect.

> Note that it is not possible to reuse the exact same converter instance configured in Kafka Connect, however we can reuse the same concepts so we do no need to reinvent the wheel

This means that the transformer can be configured using familiar concepts, like:

```properties
transforms = wasm
transforms.wasm.type             = class com.github.lburgazzoli.kafka.transformer.wasm.WasmTransformer
transforms.wasm.module.path      = src/test/resources/functions.wasm
transforms.wasm.function.name    = value_to_key
transforms.wasm.header.converter = class org.apache.kafka.connect.storage.StringConverter
transforms.wasm.key.converter    = class org.apache.kafka.connect.storage.StringConverter
transforms.wasm.value.converter  = class org.apache.kafka.connect.storage.StringConverter
```

Given such configuration, then in the `getValueFn` shown above, the function `WasmRecordConverter::fromConnectValue` would leverage the configured `value.converter` to converts the ConnectRecord's value to a byte array.
The same would apply the other way around where the `WasmRecordConverter::toConnectValue` function to convert from a byte array to ConnectRecord element.

For the complete implementation, have a look at the [WasmFunction](https://github.com/lburgazzoli/kafka-connect-wasm-transformer/blob/main/src/main/java/com/github/lburgazzoli/kafka/transformer/wasm/WasmFunction.java) implementation.

# Wrap it all up

Now that we have a _Wasm_ function and the a _Apache Kafka Connect Wasm Transformer_, we can use it in a test/example to proof it works.
Assuming that a `functions.wasm` file is available, we can then write a simple test like:


```groovy
def 'direct transformer (to_upper)'() {
    given:
        def t = new WasmTransformer()
        t.configure(Map.of(
                WasmTransformer.WASM_MODULE_PATH, 'src/test/resources/functions.wasm',
                WasmTransformer.WASM_FUNCTION_NAME, 'to_upper',
        ))

        def recordIn = sourceRecord()
                .withTopic('foo')
                .withKey('the-key'.getBytes(StandardCharsets.UTF_8))
                .withValue('the-value'.getBytes(StandardCharsets.UTF_8))
                .build()

    when:
        def recordOut = t.apply(recordIn)
    then:
        recordOut.value() == 'THE-VALUE'.getBytes(StandardCharsets.UTF_8)
    cleanup:
        closeQuietly(t)
}
```

# Conclusion

While this is merely a proof-of-concept, the integration of _Apache Kafka Connect_ with _Wasm_ holds great potential.
Stay tuned for future posts, where we'll delve deeper into the possibilities.

To learn more about this implementation and explore its code, visit my [GitHub repository](https://github.com/lburgazzoli/kafka-connect-wasm-transformer)
