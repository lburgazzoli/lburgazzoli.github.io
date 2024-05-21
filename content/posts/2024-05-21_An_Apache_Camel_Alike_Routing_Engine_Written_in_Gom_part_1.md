---
title: Apache Camel alike routing engine written in GoLang pt. 1'
date: 2024-05-21
draft: false
tags:
- go
- wasm
- integration
---

I recently had some time to continue exploring how to combine some of the technologies I had on my radar for quite a while and I finally got something that - even if it is just a proof of concept / experiment - can finally be shown.

What we will go through in this post is:
- Apache Camel alike routing engine written in GoLang
- Apache Camel K alike controller
- Embedded WebAssembly engine for extensible and safe message routing and transformation
- Actors Model
- OCI Artifacts

**_Notes:_** 
- **_the result of this work is by no mean expected to land in the official Apache Camel project or in any Red Hat Integration product_** 

# Background

To get started, let’s have a basic understanding of the components/technologies we are going to mention in this post:

- [**Apache Camel**](https://camel.apache.org/): an open-source integration framework that provides a set of tools and patterns for facilitating the integration of various applications, systems, and technologies. It simplifies the process of connecting different software components and systems, allowing them to communicate and exchange data in a flexible and efficient manner. 

- [**WebAssembly (Wasm)**](https://webassembly.org/): a low-level bytecode format designed as a portable target for the compilation of high-level languages like C, C++, and Rust, enabling deployment on the web for client and server applications. Some of the key advantages of Wasm are:
  - **performances**: contrary to something like JavaScript, Wasm is a compiled language, meaning that it can be executed more quickly and efficiently. 
  - **portability**: since it is a bytecode format, Wasm can be compiled from multiple high-level languages and run on any platform that has a Wasm runtime, such as web browsers, servers or even IoT devices.
  - **security**: Wasm is designed to be executed in a sandboxed environment, ensuring safety and security
- [**Actors Model**](https://en.wikipedia.org/wiki/Actor_model): a model of concurrent computation for developing parallel and distributed systems. Each actor is an autonomous object that operates concurrently and asynchronously, receiving and sending messages to other actors, creating new actors, and updating its own local state. An actor system consists of a collection of actors, some of whom may send messages to, or receive messages from, actors outside the system

- [**OCI Artifacts**](https://github.com/opencontainers/artifacts) Container registries were originally designed to store and distribute container images, but software engineers soon saw their utility for storing non-image objects such as Helm charts, Tekton bundles, and policy modules. **OCI artifacts** are a way of using OCI registries, or container registries that are compliant with specifications set by the [**Open Container Initiative**](https://opencontainers.org/), to store arbitrary files. By storing these objects in the same infrastructure as their containers, software engineers are able to consolidate, for example, their security and management efforts. 

Let’s also define what acronyms stands for:
- **EIP**: [Enterprise Integration Pattern ](https://www.enterpriseintegrationpatterns.com/)
- **DSL**: Domain Specific Language

# Goal

Provide a lightweight and safe cloud native routing and mediation engine built with modern technologies.

# An Apache Camel alike routing engine written in Go Lang

_Note: I assume that the reader has a minimal knowledge of the Apache Camel architecture, if not some info can be found_ [_here_](https://camel.apache.org/manual/architecture.html) _so, for the rest of the post, I’ll focus on the parts that are more relevant for my POC._

It is not the first time that a port of Apache Camel to Go lang is being discussed, for example some friend of mine started [Gamel](https://github.com/ugol/gamel) (note, I like the name a lot so if the original author agrees, I may use it at some point), however the project did not progress much. 

I also created my own experimental project [camel-go v1 ](https://github.com/lburgazzoli/camel-go/tree/v1)but I had not much time to move it forward till some months ago when I started working on it again but looking at the goal from a different perspective and I rewrote it by using Apache Camel as an inspiration more than something to merely port to another language. 

When talking about Apache Camel, people often mention the large set of connectivity options the project provides, however the key feature of Apache Camel is the implementation of [EIP](https://camel.apache.org/components/4.0.x/eips/enterprise-integration-patterns.html) and the [DSL](https://camel.apache.org/manual/dsl.html) it offers for defining integration routes and processing logic in a human-readable and expressive manner, abstracting away much of the boilerplate code required for integration tasks. 

One of the key aspects of my POC is then to provide a similar UX/DX, focusing in particular on the [YAML DSL](https://camel.apache.org/components/4.0.x/others/yaml-dsl.html) as the primary way of defining EIPs and Routes.

Let’s start with a simple example:

```yaml
- route:
   from:
     uri: 'kafka:iot'
     steps:
       - to: 'log:in'
       - choice:
           when:
             - jq: 'header("kafka.KEY") == "sensor-1"'
               steps:
               - transform:
                   jq: '.foo'
               - to: 'log:s1'
             - jq: 'header("kafka.KEY") == "sensor-2"'
               steps:
                 - transform:
                     jq: '.bar'
                 - to: 'log:s2
           otherwise:
             steps:
               - to: "log:unknown"
```

At high level, in Apache Camel, each route is translated into a series of functions ([processors](https://camel.apache.org/manual/processor.html) in Camel jargon) each of which implements a specific EIP, that form an execution pipeline. When an event triggers the execution of a route, then each function is submitted by the pipeline to the [internal executor engine](https://camel.apache.org/manual/threading-model.html) for execution in a thread pool. 

As the new engine is written in go, we can now leverage go’s concurrency building blocks (goroutines, channels) to implement a similar model. We can even go a little bit further by also implementing the routing engine as an actor system (at this stage, the engine leverages the excellent [Proto.Actor](https://github.com/asynkron/protoactor-go) library as the foundation).


# Actor Model

Why actors ? 

Because actors have some characteristics that make them very suitable and interesting in our use case, to mention some:

- **State**: represents the internal state of the actor and depends on the specific actor, some actor are stateless but some may requires to store some state (i.e. throttling messages)
- **Behavior**: the actions to be taken in reaction to the message at that point in time.
- **Mailbox**: it connects sender and receiver; each actor has exactly one mailbox to which all senders enqueue their messages
- **Children:** an actor can create children for delegating sub-tasks, in such cases the actor will automatically supervise them.
- **Supervisor:**  the supervisor delegates tasks to subordinates and therefore must respond to their failures. When a subordinate detects a failure (i.e. it panics), it sends a message to its supervisor, signaling failure**.** By default child actors forward the error to their supervisor till the global one is reached.

In addition to the common features of actors, the Prto.Actor library provides some additional features (not yet part of the POC):
- **Middleware**: allow to intercept incoming and outgoing messages and add some specific behavior such as tracing, metrics and logging.
- **Location Transparency**: all interactions of actors use purely message passing and everything is asynchronous, so all functions are/should be available equally when running within a single actor system or on a cluster of machines making it possible i.e. to schedule some processing logic according to data affinity rule 
- **Persistence:** in some scenarios it is needed or highly desirable to permanently save an actor's state so that it can be restored upon actor restart making the system able to recover from exactly where it left.

That said, the route introduced in the previous section can be described by a network of actors, each of them implementing a specific EIP or function:

![actor-engine](/images/actor-engine.svg)

When a route is loaded in the Camel Go runtime, then the runtime creates a root actor which is often described by a _from_ definition, which then spawns all its child actors which themself can spawn other children. 

This tree of actors is called an _actor system._

Each parent knows about the child actor(s), and has access to the child’s address so a parent actor can send messages to a child actor. Child actor(s) know who is the sender of every message they receive and will reply back, always in the form of a message, to the sender once the message is processed.

One important characteristic of an actor system is that because actors can only communicate through messages and that they rely on a mailbox which ensures that only one message is processed at a time, actors can operate in a single-threaded illusion so the state of the actor is protected against any of the normal concerns of concurrency.


# Wasm for extensibility

I’ve been working for some time toward providing a managed connectivity service for Apache Kafka and one of the most critical yet difficult parts is how to allow running non-trivial processing logic in a simpler and safer way. Over time we experimented with a number of options such as functions, scripting languages, custom images, etc however none was really satisfying, i.e. 

- scripting languages:
  - Require the users to eventually learn a new language
  - The execution may harm the host system as scripts can access files, environments and other host resources
- functions:
  - Makes the deployment of applications more complicated as it requires to have other resources deployed together (the functions)
  - Costs is often higher as data must be transferred between the main application and the functions which leads to I/O and often escapes data locality  
  - Failure handling become more complicated 

So I decided to give Wasm a try as Wasm is meant to be:
- **Polyglot**: A number of languages support Wasm as a compilation target which makes it possible to attract a larger number of developers that may not be familiar with the language the runtime is written with.
- **Secure**: one of WebAssembly's main goals is to execute untrusted code in a safe manner inside of a sandbox where only the host can configure what the code running in the sandbox has access to, making it a perfect fit for plugins/extensions. 
- **Embeddable:** a Wasm runtime can be embedded in the host application, allowing the application to be securely extended with any language that can be compiled to Wasm without requiring any additional infrastructure and without having the data leaving the application.

There are a number of Wasm runtimes out there but since Go is the language of choice for this POC, I leveraged [Wazero](https://wazero.io) as it is the only zero dependency WebAssembly runtime (i.e. it does not require any native library binding). 

At this stage the pseudo signature for message processing expected by the runtime is:

```go
func (inOut Message) error
```

Albeit being a very simple function, invoking it from the host program is not trivial as you need to cross the host/guest memory boundaries which can be done in a number of ways among which:

1. By manually dealing with memory allocation and deallocation within WASM linear memory

2. By using STDIN/OUT as a way to exchange data (CGI anyone)

My first attempt was to use option **A** which led me to do a very long research to understand how to safely manage memory between host and guest in particular in relation to languages such as Go that have a garbage collector. Some results can be seens by looking at the [allocation examples](https://github.com/tetratelabs/wazero/tree/main/examples/allocation) on the wazero repo (thx to Adrian Cole and Edoardo Vacchi for the patience and guidance) but for the sake of simplicity (remember, this is just a POC) and portability I then decided to move toward options **B** which in the host side ended up being similar to the example here:

```go
func (p *Plugin) invoke(in any, out any) error {
   fn := p.lookupFunction("process")
   if fn == nil {
       return nil, errors.New("process is not exported")
   }

   data, err := json.Marshal(in)
   if err != nil {
       return err
   }

   // clean up the buffer
   p.stdin.Reset()
   p.stdout.Reset()

   defer func() {
       // clean up the buffer when the method
       p.stdin.Reset()
       p.stdout.Reset()
   }()

   ws, err := p.stdin.Write(data)
   if err != nil {
       return err
   }

   // invoke the function with the size of the message
   // so the guest knows how many bites have to be read
   // from STDIN
   ptrSize, err := fn.Call(context.Background(), uint64(ws))
   if err != nil {
       return err
   }

   // since WASM virtual machine supports only 32 bits
   // we can use 32 bit to hold the response data size
   // and the remaining for flags, i.e. to indicate
   // that an error has occurred
   resFlag := uint32(ptrSize[0] >> 32)
   resSize := uint32(ptrSize[0])

   bytes := make([]byte, resSize)
   _, err = p.stdout.Read(bytes)
   if err != nil {
       return err
   }

   switch resFlag {
   case 1:
       return errors.New(string(bytes))
   default:
       return json.Unmarshal(bytes, &out)
   }
}
```

To ease the process of writing processor a small SDK has been implemented:

```go
type Processor func(context.Context, *Message) (*Message, error)

var processor Processor

func RegisterProcessors(p Processor) {
   processor = p
}

//export process
func _process(size uint32) uint64 {
   b := make([]byte, size)

   _, err := io.ReadAtLeast(os.Stdin, b, int(size))
   if err != nil {
       return 0
   }

   req := Message{}
   if err := json.Unmarshal(b, &req); err != nil {
       return 0
   }
   res, err := processor(context.Background(), &req)
   if err != nil {
       n, err := os.Stdout.WriteString(err.Error())
       if err != nil {
           return 0
       }

       // Indicate that this is the error string
       return (uint64(1) << uint64(32)) | uint64(n)
   }

   b, err = json.Marshal(res)
   if err != nil {
       return 0
   }

   n, err := os.Stdout.Write(b)
   if err != nil {
       return 0
   }

   return uint64(n)
}
```

Which can be then leverage to write a processor without having to deal with serialization/deserialization and/or allocations: 

```go
func main() {
   // register the processor function
   RegisterProcessors(Process)
}

func Process(_ context.Context, r *Message) (*Message, error) {
   request.Data = []byte(strings.ToLower(string(r.Data)))
   return request, nil
}
```

To use a Wasm function in the routing engine, then we can leverage the _wasm language,_ as an example

```yaml
- route:
   from:
     uri: 'timer:foo?period=1s'
     steps:
       - transform:
           wasm: 'etc/wasm/fn/to_upper.wasm'
       - to:
           uri: "log:info"
```

# OCI Artifacts for Wasm distribution

Now that we have a somehow working Routing Engine (**NOTE:** **only few EIP are supported at this stage**) which supports Wasm as a way to implement transformation logic, we must define how to make it easy for users to ship and consume Wasm artifacts.

There are of course many way of doing it like building custom container images with the compiled Wasm models includes or make the Wasm modules available on a filesystem that the engine can read from however, since pretty much every cloud-native systems has to deal with Container Image registries, we could leverage OCI Registries and OCI Artifacts. 

So, what is an OCI Artifact ?

It is an ongoing [Open Container Initiative](https://opencontainers.org/) initiative to define a spec which would allow OCI Registries to store arbitrary files. This is not something completely new and a number of projects have already started using OCI Artifacts, as example:

- [Web Assembly Hub](https://www.solo.io/products/web-assembly/)
- [Use OCI-based registries](https://helm.sh/docs/topics/registries/)
- [GitHub Packages Container registry is generally available](https://github.blog/2021-06-21-github-packages-container-registry-generally-available/)
- [sigstore/cosign: Container Signing](https://github.com/sigstore/cosign#oci-artifacts)
- [OCI cheatsheet | Flux](https://fluxcd.io/flux/cheatsheets/oci-artifacts/)

In our case we leverage the [ORAS](https://oras.land/) project to make it easy to package Wasm modules as an OCI Artifact so all we need to do is to set the right media type to let the Routing Engine to identify the layers that provide Wasm modules, as an example:

```
oras push \
    quay.io/lburgazzoli/camel-go-wasm:latest \
    etc/wasm/fn/to_upper.wasm:application/vnd.module.wasm.content.layer.v1+wasm
```

This command would push the _simple\_process.wasm_ module file to quay.io (a compatible OCI Registry) with the media-type expected by the Go Routing Engine. The layer in which the wasm module is stored is then named after the file path so it will be _etc/wasm/fn/simple\_process.wasm_.

To use the OCI Artifact, it is enough to instruct the _wasm language_ to lookup the module form an image:

```yaml
- route:
   from:
     uri: 'timer:foo?period=1s'
     steps:
       - transform:
           wasm:
             image: 'quay.io/lburgazzoli/camel-go-wasm:latest'
             path: 'etc/wasm/fn/yo_upper.wasm'
       - to:
           uri: "log:info"
```

At this point the Camel Go’s _wasm_ language inspects the configured container image and then downloads and loads the layer containing the configured Wasm module.

# Conclusion

In this first part, I went into some of the details about the implementation of an Apache Camel-like routing engine written in GoLang. that leverages Wasm for extensibility. In the second part, I’m going through some more detail about deployment options.    
