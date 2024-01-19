---
title: "Thoughts on WIT and wit-bindgen"
date: 2023-05-03T00:00:00-00:00
description: Discussion of (current) ergonomics of using WIT for API description.
draft: true
tags:

- Rust
- Wasm

---

The Wasm component model uses WIT to define runtime APIs available to guest 
components. `wit-bindgen` can then be used by both guest component authors
and host platforms to generate bindings in their language of choice to
consume or implement these runtime APIs respectively. This post discusses 
some of the rough edges encountered when taking this for a spin. 

<!--more-->

First, I would like to note that many of these design choices were carefully
considered by the contributors to the WIT specification and `wit-bindgen`. 
This post reflects a very shallow understanding of these decisions and further
discussion may be found in issues in both the
[bytecodealliance/wit-bindgen](https://github.com/bytecodealliance/wit-bindgen) and 
[WebAssembly/component-model](https://github.com/WebAssembly/component-model) repositories.

# Param and Return Types

Prior to `v0.6`, `wit-bindgen ` would generate types in different ways depending on
if they were arguments to host or guest functions. Consider the following WIT definition
for a "proxy" component:

```wit
interface http {
    record request {
        url: string,
        body: option<string>
    }
    record response {
        body: string
    }

    // Proxy component can make a new outbound request
    fetch: func(req: request) -> response
}

default world proxy {
    use self.http.{response, request}
    
    // Invoke the proxy component
    export fetch: func(request) -> response
}
```

The results in roughly the following Rust code for the component:

```rust
impl Proxy for Guest {
    fn fetch(request: http::RequestResult) -> http::Response {
        http::fetch(http::RequestParam {
            url: &request.url,
            body: request.body.as_deref()
        })
    }
}
```

Note that we cannot forward what should be the same type to the host `fetch` function,
and mapping the two types is quite clunky for non-primitive fields due to one using
references.

This was resolved by default in a recent PR, see this
[discussion](https://github.com/bytecodealliance/wit-bindgen/issues/551). With this
change, you can now do this!

```rust 
impl Proxy for Guest {
    fn fetch(request: http::Request) -> http::Response {
        http::fetch(&request)
    }
}
```

# Map and Recursive Types

Types like [serde_json::Value](https://docs.rs/serde_json/latest/serde_json/value/enum.Value.html)
are very useful for constructing arbitrary data structures which could
represent complex arguments to host APIs or request and response bodies.
Such a type requires both maps and self-referential types. Ideally you
could do something like this:

```wit
interface types {
    variant value {
        null(void),
        bool(bool),
        number(f64),
        string(string),
        array(list<value>),
        object(map<string, value>)
    }
}
```

Map types across languages are more complex than the other primitive
types that WIT supports, so it makes sense that they would not be
straightforward to support, but most languages do have some sort of 
map type, and it is currently very limiting to instead have to
represent such data as `list<tuple<string, string>>`.

Recursive types are even more difficult to support, as some languages
may not have them at all, but they are very useful for
representing complex data structures such as trees. For more context,
see this [discussion](https://github.com/WebAssembly/component-model/issues/56).

# WASI Linking

Compiling a component to `wasm32-wasi` is currently tricky. Hosts implement
prototype Wit definitions with Wasi functionality, such as
[this example](https://github.com/bytecodealliance/preview2-prototyping/tree/main/wit).
A shim component built from the same repository is then included in the
component binary as an "adapter". This shim consumes the new Wit-based
Wasi APIs, and exports the legacy Wasi APIs that the `wasm32-wasi`
compiler target expects. 

# Objects

Another major ergonomic gap currently is the lack of objects. Such a type would
be like the `record` or `variant` type, but with host-implemented methods
that operate on specific instances of the type. This would be very useful for
runtimes to provide non-primitive types, for instance:

```
object request {
    body: string

    // Host implements complexities of JSON parsing
    json: func() -> value
}
```

Another appealing advantage of such types is that they could significantly reduce
the amount of data shared between components. This is because components can
pass references between them with methods to interact safely with the contained data,
instead of fully copying the data.

It turns out that this _is_ planned in the WIT spec as `resource` types, but
this is currently [vaguely documented](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md#handles),
and is not implemented yet in `wit-bindgen`.

Some experimental codebases using Wit have implemented a workaround which involves
creating handles which reference objects in a global table managed by the host. Guests
then explicitly pass these handles into host methods. A description of this approach can
be found [here](https://github.com/WebAssembly/WASI/blob/main/docs/WitInWasi.md#Resources).


