---
title: "Detecting Undefined Behavior in Rust with Miri in GitHub Actions"
date: 2024-06-01
draft: false
description: "Learn how to use Miri in CI to detect various types of errors in your Rust codebase and how to set up GitHub Actions for automatic PR runs."
tags:
  - Rust
  - CI/CD
  - Miri
  - GitHub Actions
---

# Detecting Undefined Behavior in Rust with Miri in GitHub Actions

## Introduction to Miri

Miri is an interpreter for Rust's mid-level intermediate representation (MIR), which allows for the detection of undefined behavior and other errors at compile time. Integrating Miri into Continuous Integration (CI) workflows can significantly improve the quality of a Rust codebase by catching errors early in the development process. This can be especially important if your codebase requires the use of unsafe blocks of Rust code which may prevent the Rust compiler from catching bugs that you would normally expect it to.

<!--more-->

## Types of Errors Detected by Miri

You can find an extensive list of the kinds of errors that Miri can catch in [miri's test suite](https://github.com/rust-lang/miri/tree/master/tests). Note that many of these examples do not compile, and are used only to explain the bug.

### Out-of-Bounds Memory Accesses

Out-of-bounds memory accesses occur when a program tries to read or write memory outside the allocated range. This can lead to unpredictable behavior and security vulnerabilities.

#### Example

```rust
let arr = [1, 2, 3];
let _ = arr[3]; // This will cause an out-of-bounds access
```

### Use-After-Free

Use-after-free errors happen when a program continues to use a pointer after the memory it points to has been freed. This can lead to data corruption or the execution of arbitrary code.

#### Example

```rust
unsafe fn make_ref<'a>(x: *mut i32) -> &'a mut i32 {
    &mut *x
}
unsafe {
    let x = make_ref(&mut 0); // The temporary storing "0" is deallocated at the ";"!
    let val = *x; //~ ERROR: has been freed
}
```

### Invalid Use of Uninitialized Data

Using uninitialized data can lead to undefined behavior, as the contents of the uninitialized memory are unknown.

#### Example

```rust
let x: i32;
println!("{}", x); // Undefined behavior
```

### Violation of Intrinsic Preconditions

Certain operations in Rust have preconditions that must be met. Violating these preconditions can cause undefined behavior.

#### Example

```rust
unsafe {
    let x: usize = std::mem::transmute("hello");
    // Transmuting a &str to usize is undefined behavior
}
```

### Not Sufficiently Aligned Memory Accesses and References

Memory alignment errors occur when a reference does not meet the alignment requirements for the type it points to.

#### Example

```rust
let x = [0u8; 3];
let y = &x[1] as *const u8 as *const u16;
unsafe {
    let _ = *y; // Possible misaligned reference
}
```

### Violation of Basic Type Invariants

Rust's type system has certain invariants that must be upheld. Violating these invariants can lead to undefined behavior.

#### Example

```rust
let mut v = vec![1, 2, 3];
let r1 = &mut v[0];
let r2 = &mut v[1];
std::mem::swap(r1, r2); // Violates Rust's aliasing rules
```

### Memory Leaks

While not undefined behavior, memory leaks can lead to resource exhaustion and degraded performance.

#### Example

```rust
std::mem::forget(Box::new(42)); // Memory leak
```

## Setting Up GitHub Actions for Automatic PR Runs with Miri

To integrate Miri into your CI workflow with GitHub Actions, follow these steps:

1. Create a new file in your repository under `.github/workflows/miri.yml`.
2. Add the following content to `miri.yml`:

```yaml
name: Miri

on: [push, pull_request]

jobs:
  miri:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Install Rust toolchain
      uses: actions-rs/toolchain@v1
      with:
        profile: minimal
        # Miri requires nightly Rust
        toolchain: nightly
        override: true
        components: miri
    - name: Run Miri
      # Run all of your tests inside of Miri interpreter
      run: cargo miri test
      # Or, if you have a binary project, run it in Miri interpreter
      run: cargo miri run
```

3. Commit and push the changes to your repository.

This GitHub Actions workflow will automatically run Miri on every push and pull request, helping to catch errors early and maintain high code quality.

## Conclusion

Integrating Miri into CI workflows offers a powerful way to improve the quality of Rust codebases by detecting a wide range of errors. By setting up GitHub Actions for automatic PR runs with Miri, teams can ensure that their code is robust, safe, and free of common errors before merging changes.
