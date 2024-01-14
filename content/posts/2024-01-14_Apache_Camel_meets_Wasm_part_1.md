---
title: 'Apache Camel meets WASM - Part 1'
date: 2024-01-14
draft: false
tags:
- wasm
- camel
---

I recently stumbled upon a fascinating [JavaAdvent article](https://www.javaadvent.com/2023/12/chicory-wasm-jvm.html) discussing a native [WebAssembly](https://webassembly.org) runtime for the JVM called Chicory. Intrigued by the potential synergy between _WebAssembly_ and _Apache Camel_, I decided to experiment with integrating them.

# Background

To get started, let’s have a basic understanding of the components/technologies we are going to mention in this post:

- [Apache Camel](https://camel.apache.org) is an open-source integration framework that provides a set of tools and patterns for facilitating the integration of various applications, systems, and technologies. It simplifies the process of connecting different software components and systems, allowing them to communicate and exchange data in a flexible and efficient manner.
- [WebAssembly (Wasm)](https://webassembly.org) is a low-level bytecode format designed as a portable target for the compilation of high-level languages like C, C++, and Rust, enabling deployment on the web for client and server applications.
- [Chicory](https://github.com/dylibso/chicory) is a JVM native WebAssembly runtime. It allows you to run WebAssembly programs with zero native dependencies or JNI. Chicory can run Wasm anywhere that the JVM can go.

# Goals

Our primary objective is to support **Wasm** as a means to provide custom processing and transformation logic within **Apache Camel**.

# Implementation

As a first experiment, I've started by [implementing](https://github.com/lburgazzoli/camel-wasm) a dedicated [Apache Camel component](https://camel.apache.org/camel-core/working-with-camel-core/index.html#_components) for **Wasm**. If you are a little familiar with Apache Camel, then you know that a _Component_ is pretty much a factory to create _Producers_ and _Consumers_, which are entities interfacing with the target system (a remote API, a Java library, ect).

![camel-component](/images/camel-component.png)

In our case, we just want to invoke a function provided as a Wasm bytecode and since Apache Camel is written in Java, then here is where **Chicory** enters the game.
Thanks to the excellent **Chicory's Low Level API**, writing an _Producer_ becomes straighforward so, ignoring all the boilerplate required to create a _Component_, invoking a _Wasm_ function boils down to a simple class like this one:

```java
import java.util.Objects;

import com.dylibso.chicory.runtime.ExportFunction;
import com.dylibso.chicory.runtime.Instance;
import com.dylibso.chicory.runtime.Module;
import com.dylibso.chicory.wasm.types.Value;

public class WasmFunction {
    private final Module module;
    private final String functionName;

    private final Instance instance;
    private final ExportFunction function;
    private final ExportFunction alloc;
    private final ExportFunction  dealloc;

    public WasmFunction(Module module, String functionName) {
        this.module = Objects.requireNonNull(module);
        this.functionName = Objects.requireNonNull(functionName);

        this.instance = this.module.instantiate();
        this.function = this.instance.getExport(this.functionName);
        this.alloc = this.instance.getExport("alloc");
        this.dealloc = this.instance.getExport("dealloc");
    }

    public byte[] run(byte[] in) throws Exception {
        Objects.requireNonNull(in);

        int inPtr = -1;
        int inSize = in.length;
        int outPtr = -1;
        int outSize = 0;

        try {
            inPtr = alloc.apply(Value.i32(inSize))[0].asInt();
            instance.getMemory().write(inPtr, in);

            Value[] results = function.apply(Value.i32(inPtr), Value.i32(inSize));
            long ptrAndSize = results[0].asLong();

            outPtr = (int)(ptrAndSize >> 32);
            outSize = (int)ptrAndSize;

            return instance.getMemory().readBytes(outPtr, outSize);
        } finally {
            if (inPtr != -1) {
                dealloc.apply(Value.i32(inPtr), Value.i32(inSize));
            }
            if (outPtr != -1) {
                dealloc.apply(Value.i32(outPtr), Value.i32(outSize));
            }
        }
    }
}
```

As usual the devil is in the detail.

## Memory Management and Data Exchange

In _Wasm_, sharing objects between the host, in this case the _JVM_, and the _Wasm_ module is deliberately restricted, so the parties must agree on a number of things, among which:
1. the schema of the data exchnaged
2. the memory management, so how data must be allocated and when/how/who it should be de-allocate 

The data structure chosen for this POC is a subset of an Apache Camel Exchange, containing headers and a body encoded as a base64 string.

```java
public static class Wrapper {
    @JsonProperty
    public Map<String, String> headers = new HashMap<>();

    @JsonProperty
    public byte[] body;
}
```

To be sure the data is valid for the entire execution of the Apache Camel's processor, the host is the one responsible to de-allocate memory and this has an impact on how the guest function is implemented. 

For this POC, I decided to use [Rust](https://www.rust-lang.org/) to write the _Wasm_ function and below you can find exemplare allocation and deallocation functions:

```rust
pub extern "C" fn alloc(size: u32) -> *mut u8 {
    let mut buf = Vec::with_capacity(size as usize);
    let ptr = buf.as_mut_ptr();

    // tell Rust not to clean this up
    mem::forget(buf);

    ptr
}

pub unsafe extern "C" fn dealloc(ptr: &mut u8, len: i32) {
    // Retakes the pointer which allows its memory to be freed.
    let _ = Vec::from_raw_parts(ptr, 0, len as usize);
}
```

Now we can implement the real function which ins this case does nothing more that take the input body and make it upper case:

```rust
#[derive(Serialize, Deserialize)]
struct Message {
    headers: HashMap<String, serde_json::Value>,

    #[serde(with = "Base64Standard")]
    body: Vec<u8>,
}

pub extern fn process(ptr: u32, len: u32) -> u64 {
    let bytes = unsafe {
        slice::from_raw_parts_mut(
            ptr as *mut u8,
            len as usize)
    };

    let mut msg: Message = serde_json::from_slice(bytes).unwrap();
    msg.body = String::from_utf8(msg.body).unwrap().to_uppercase().as_bytes().to_vec();

    let out_vec = serde_json::to_vec(&msg).unwrap();
    let out_len = out_vec.len();
    let out_ptr = alloc(out_len as u32);

    unsafe {
        std::ptr::copy_nonoverlapping(
            out_vec.as_ptr(),
            out_ptr,
            out_len as usize)
    };

    return ((out_ptr as u64) << 32) | out_len as u64;
}
```

So what the function does is:
1. read the chunk of memory the host function has filled at a given `ptr`
2. de-serialize the buffer to a type that implements the agreed schema
3. apply the logic
4. serialize it back to a buffer
5. return to the host the `ptr` where the result can be found

# Wrap it all up

Now that we have a _Wasm_ function and the a _Apache Camel Component for Wasm_, we can use it in a test/example to proof it works so assuming that a `functions.wasm` file is available in the classpath, then we can write something like:

```java
try (CamelContext cc = new DefaultCamelContext()) {
    FluentProducerTemplate pt = cc.createFluentProducerTemplate();

    cc.addRoutes(new RouteBuilder() {
        @Override
        public void configure() throws Exception {
            from("direct:in")
                .to("wasm:process?resource=functions.wasm");
        }
    });

    cc.start();

    Exchange out = pt.to("direct:in")
        .withHeader("foo", "bar")
        .withBody("hello")
        .request(Exchange.class);


    assertThat(out.getMessage().getHeaders())
        .containsEntry("foo", "bar");
    assertThat(out.getMessage().getBody(String.class))
        .isEqualTo("HELLO");

}
```

# Conclusion


While this is merely a proof-of-concept, the integration of _Apache Camel_ with _Wasm_ holds great potential. 
Stay tuned for future posts, where we'll delve deeper into the possibilities

To learn more about this implementation and explore its code, visit my [GitHub repository](https://github.com/lburgazzoli/camel-wasm)
