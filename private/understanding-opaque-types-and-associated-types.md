## Introduction

Rust is a systems programming language known for its safety and concurrency features. Among its many advanced type system capabilities are opaque types and associated types, which play crucial roles in ensuring code abstraction and flexibility. One day, I spent quite some time figuring out why the following code wouldn't compile: 

```rust
fn iterator() -> impl Iterator<Item = u32> {
    vec![1, 2, 3].into_iter()
}

trait Foo {
    fn iterator(&self) -> impl Iterator<Item = u32>;
}

struct Struct;

impl Foo for Struct {
    fn iterator(&self) -> impl Iterator<Item = u32> {
        vec![4, 5, 6].into_iter()
    }
}

fn main() {
    let mut array = vec![iterator()];
    
    let x = Struct;
    array.extend(x.iterator());
}
```

You can check this code at [playground].

Forget about what is Iterator for now: the code seems to be fine because both `iterator` method returns the type of implementing `Iterator`. However, it is actually incorrect. 
The compiler raises an error for this code.

```bash
error[E0271]: type mismatch resolving `<impl Iterator<Item = u32> as IntoIterator>::Item == impl Iterator<Item = u32>`
   --> src/main.rs:21:18
    |
1   | fn iterator() -> impl Iterator<Item = u32> {
    |                  ------------------------- the expected opaque type
...
21  |     array.extend(x.iterator());
    |           ------ ^^^^^^^^^^^^ expected opaque type, found `u32`
    |           |
    |           required by a bound introduced by this call
    |
    = note: expected opaque type `impl Iterator<Item = u32>`
                      found type `u32`
note: the method call chain might not have had the expected associated types
   --> src/main.rs:21:20
    |
20  |     let x = Struct;
    |             ------ this expression has type `Struct`
21  |     array.extend(x.iterator());
    |                    ^^^^^^^^^^ `IntoIterator::Item` is `u32` here
note: required by a bound in `extend`
   --> /playground/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/iter/traits/collect.rs:415:31
    |
415 |     fn extend<T: IntoIterator<Item = A>>(&mut self, iter: T);
    |                               ^^^^^^^^ required by this bound in `Extend::extend`

For more information about this error, try `rustc --explain E0271`.
```

The issue arises because each iterator method returns different types. This discrepancy is why the code does not compile. In this blog, we will explore the differences between these two concepts and understand their respective use cases.


[playground]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn+iterator%28%29+-%3E+impl+Iterator%3CItem+%3D+u32%3E+%7B%0A++++vec%21%5B1%2C+2%2C+3%5D.into_iter%28%29%0A%7D%0A%0Atrait+Foo+%7B%0A++++fn+iterator%28%26self%29+-%3E+impl+Iterator%3CItem+%3D+u32%3E%3B%0A%7D%0A%0Astruct+Struct%3B%0A%0Aimpl+Foo+for+Struct+%7B%0A++++fn+iterator%28%26self%29+-%3E+impl+Iterator%3CItem+%3D+u32%3E+%7B%0A++++++++vec%21%5B4%2C+5%2C+6%5D.into_iter%28%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+array+%3D+vec%21%5Biterator%28%29%5D%3B%0A++++%0A++++let+x+%3D+Struct%3B%0A++++array.extend%28x.iterator%28%29%29%3B%0A++++%0A++++println%21%28%22%7B%7D%22%2C+array%29%3B%0A%7D%0A

## Understanding Opaque Types

Opaque types in Rust allow you to hide the implementation details of a type, providing a level of abstraction. This is useful for encapsulating functionality and ensuring that users of your type cannot rely on its internal structure.

Let's revisit the function from the introduction

```rust
fn iterator() -> impl Iterator<Item = u32> {
    vec![1, 2, 3].into_iter()
}
```

Here, iterator returns an `impl Iterator<Item = u32>`. The impl Trait syntax is used to specify that the function returns some type that implements the Iterator trait with Item = u32, without exposing the actual type. This is an example of an opaque type.

Using `impl Trait` in function return positions helps to hide the concrete type of the iterator, enabling you to change the underlying type without affecting code that depends on the function. However, the limitation is that each function returning an `impl Trait` is considered to have a distinct, unnamed type, even if the trait and the associated types are the same.

Opaque types are great for:

1. **Encapsulation**: Hiding implementation details and exposing only necessary interfaces.
2. **Flexibility**: Allowing changes to the underlying implementation without breaking dependent code.

## Understanding Associated Types

Associated types are a way to define a placeholder type inside a trait that can vary depending on the implementation. They provide a means to define relationships between types in a trait, making traits more flexible and expressive.

```rust
trait Foo {
    fn iterator(&self) -> impl Iterator<Item = u32>;
}
```

In this trait, the `iterator` method is defined to return an opaque type that implements `Iterator<Item = u32>`. However, because `impl Trait` in return positions creates unique, unnamed types, each implementation of `Foo` that returns a different concrete type will be seen as returning a different type. This is what caused the type mismatch error when trying to extend the vector with iterators from different sources.

### Why Use Associated Types?

Associated types are particularly useful for:

1. **Trait Flexibility**: Trait Flexibility: Allowing traits to specify relationships between types without needing generic parameters everywhere.
2. **Clearer Syntax**: Reducing the verbosity of generics in trait definitions and method signatures.
3. **Type Safety**: Ensuring that implementations of traits are consistent in their use of types.

## Key Differences

|Feature|Opaque Types|Associated Types|
|Purpose|Encapsulation and hiding implemention details|Defining type relationships within traits|
|Usage|Module and struct-level abstraction|Trait-level abstraction|
|Example Context|Implementing APIs with hidden internals|Traits with type parameters|
|Flexibility|Less flexible, more about hiding details|More flexible, allows varing types per implementation|

## Use Cases

Opaque types are best suited for scenarios where you want to hide complex internal details and provide a simple API. Associated types are ideal for generic programming where the type relationships need to be defined and varied based on implementations.

## How to resolve the problem?

Instead of using impl Trait in the trait method return type, we can define an associated type in the trait:

```rust
trait Foo {
    type Iter: Iterator<Item = u32>;

    fn iterator(&self) -> Self::Iter;
}

struct Struct;

impl Foo for Struct {
    type Iter = std::vec::IntoIter<u32>;

    fn iterator(&self) -> Self::Iter {
        vec![4, 5, 6].into_iter()
    }
}

fn iterator() -> impl Iterator<Item = u32> {
    vec![1, 2, 3].into_iter()
}

fn main() {
    let mut array = vec![iterator()];

    let x = Struct;
    array.extend(x.iterator());

    println!("{:?}", array); // Outputs the combined iterators' elements
}
```

In this updated version, `Foo` defines an associated type `Iter`, which must implement `Iterator<Item = u32>`. Each implementation of `Foo` specifies the concrete type for `Iter`. This ensures that the `iterator` method of any `Foo` implementation returns a type that can be uniformly used.

## Practical Examples

Consider a scenario where you are building a library for data processing. Opaque types can be used to hide the complex state and operations inside the library, while associated types can define how different data structures interact with the library's functions.

## Conclusion

Understanding the differences between opaque types and associated types is crucial for mastering Rust's type system. Each has its own strengths and use cases, and knowing when to use which can greatly enhance the design and flexibility of your code.
