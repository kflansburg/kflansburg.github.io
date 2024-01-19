---
title: "WIT Resources"
date: 2023-08-05T00:00:00-00:00
description: Introduction to "resources" in WIT.
draft: true
tags:

- Rust
- Wasm

---

Support for "resources" [just landed](https://github.com/bytecodealliance/wit-bindgen/commit/62f3f649e38ed4f111729595ce32ee0308ab0f7c)
in the `wit-bindgen` repository. This post explores why resources are crucial
for writing rich, object-oriented APIs in WIT.

<!--more-->

The WIT format definition [describes resources](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md#item-resource) as an "abstract type... which is an entity with a lifetime that can only be passed around indirectly via handle values." This allows host runtimes to define an interface, and then return objects
to the guest by reference (handle). These objects live in and are managed by the host, which significantly reduces the need
for copying data in and out of WASM memory.

## Motivation

As an example, before resources, it was common for HTTP `fetch` calls to return either a full copy of the response, or a "handle" to a response object. Rather than this being a first class object, it was simply an integer handle, and the end-user was required to pass this handle
into "instance methods" to interact with this object:

```rust
let response = fetch("https://kflansburg.com");

let status = response_status(response);

let body = response_body(response);
```

As you can see, this works fine, but is clunky, and makes raw host APIs unpleasent to interact with. 
Many platforms then have to ship an "sdk" or other language-specific wrapper to make this more ergonomic.

## Usage

Now, with resources, we can do this:

```rust
let response = fetch("https://kflansbug.com");

let status = response.status();
let body = response.body();
```

In this case, the `response` object can be used like a first-class object, and data such as `body` is
only copied into the guest memory if it is used.

## Definition

On the WIT side of things, resources are defined roughly (this is untested) like:

```
resource response {
    constructor(body: list<u8>, status: u16)
    status: func(self: borrow<response>) -> u16
    body: func(self: response) -> list<u8>
    
    not-found: static func() -> reponse
}

interface http {
    fetch: func(url: string) -> response
}
```

There are a couple of interesting things here. First, there is a `constructor` definition, which allows the guest to create
new response objects (which are still managed by the host), and obtain a handle to them.

Next, the `status` method takes a `borrow` of `response`, which means that response will live beyond usage of that method. `body`, on the other hand, takes ownership of and consumes `response`, which cannot be used again.

Finally, `not-found` is a `static` method, which does not take a `self` parameter, but allows for related, non-instance functions to be namespaced within the resource.

## Conclusion

Resources were a key missing piece and will signifigantly improve the ergonomics and sophistication of WASM host APIs. Specifically, APIs related to managing file descriptors, streams, or deeply nested types will be significantly easier
to develop with for both the platform and the guest. Finally, this type of pattern will be very useful for enabling
capability-based access patterns in guest components.
