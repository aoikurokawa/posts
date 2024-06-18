## Introduction
Polymorphism is a fundamental concept in object-oriented programming that allows a single interface to represent different underlying forms (data types). In Rust, polymorphism is typically achieved using traits, which are similar to interfaces in other languages. Traits enable us to define shared behavior that different types can implement, allowing for flexible and reusable code.

However, while traits are a powerful feature of Rust, they are not the only way to achieve polymorphism. After reading this [blog], Enums might be the another way to achieve polymorphism for certain scenarios. Enums allow us to define a type by enumerating its possible variants, each of which can hold different data and have different behaviors.

In this blog post, we'll explore how to implement polymorphism in Rust using enums instead of traits. We'll start with a brief overview of how traits are typically used for polymorphism, and then dive into how enums can be used as an alternative approach. We'll compare the two methods, discuss their advantages and drawbacks, and provide practical examples to illustrate the concepts.

By the end of this post, you will have a deeper understanding of how to leverage enums for polymorphism in Rust, and you might find that they offer a cleaner and more efficient solution for certain types of problems. Let's get started!

[blog](https://www.mattkennedy.io/blog/rust_polymorphism/)

## JavaScript

Before diving into Rust, let's consider a common scenario in JavaScript.
Imagine you have an interface `SlotSource` that defines a method `getSlot`. Various classes implements this interface, each providing their own version of the `getSlot` method:
```js
export interface SlotSource {
	getSlot(): number;
}

export class SlotSubscriber implements SlotSource {
	public getSlot(): number {
		...
	}
}

export class OrderSubscriber implements SlotSource {
	public getSlot(): number {
		...
	}
}
```

Now let's say we have a class `FooSubscriber` that has property `slotSource` which can be any object that implements the `SlotSource` interface. This allows `FooSubscriber` to call the `getSlot` method without needing to know the specific type of `slotSource`:
```js
export class FooSubscriber {
	slotSource: SlotSource;

	constructor(slotSource: SlotSource) {
		this.slotSource = slotSource;
	}

	public some_function(): void {
		let slot = this.slotSource.getSlot();
		// Use the slot number
	}
}
```

This approach, using interfaces and polymorphism, is common in many object-oriented languages like JavaScript. But what if we want to achieve similar polymorphic behavior in Rust?

## Traits and Polymorphism in Rust

If we want to convert following code into Rust, we can write something like this.
Normally, `interface` in JS can be translated into `trait` in Rust. 
```rs
trait SlotSource {
	fn get_slot(&self) -> u64;
}
```

Define each struct `SlotSubscriber` and `OrderSubscriber`, and then implements `SlotSource` for both structs.

```rs
struct SlotSubscriber;

impl SlotSource for SlotSubscriber {
	fn get_slot(&self) -> u64 {
		1
	}
}

struct OrderSubscriber;

impl SlotSource for OrderSubscriber {
		fn get_slot(&self) -> u64 {
			2
		}
}
```

Define `FooSubscriber`. 

```rs
struct FooSubscriber<T: SlotSource> {
	slot_source: T;
}

impl<T: SlotSource> FooSubscriber<T> {
	pub fn new(slot_source: T) -> Self {
		Self {
			slot_source
		}
	}

	pub fn some_function(&self) {
		let slot = self.slotSource.getSlot();
		println!("Slot: {slot}");
	}
}
```

Here's how you might use these structs and traits.

```rs
fn main() {
    let slot_subscriber = SlotSubscriber;
    let order_subscriber = OrderSubscriber;

    let foo1 = FooSubscriber::new(slot_subscriber);
    let foo2 = FooSubscriber::new(order_subscriber);

    foo1.some_function();
    foo2.some_function();
}
```

## Polymorphism with Enums

While traits are a common way to achieve polymorphism in Rust, we can also use enums for certain cases. Enums in Rust allow us to define a type that can be one of several variants, each of which can hold different data and have different behaviors. This can be particularly useful when the set of possible types is fixed and known in advance.

Let's revisit our example using enums instead of traits.

First, we'll define an enum SlotSource with two variants: SlotSubscriber and OrderSubscriber. Each variant will implement a method get_slot.

```rs
enum SlotSource {
    SlotSubscriber,
    OrderSubscriber,
}

impl SlotSource {
    fn get_slot(&self) -> u64 {
        match self {
            SlotSource::SlotSubscriber => {
                // Implementation for SlotSubscriber
                1
            }
            SlotSource::OrderSubscriber => {
                // Implementation for OrderSubscriber
                2
            }
        }
    }
}
```

Next, we'll define FooSubscriber to hold a SlotSource enum instead of a generic type.

```rs
struct FooSubscriber {
    slot_source: SlotSource,
}

impl FooSubscriber {
    pub fn new(slot_source: SlotSource) -> Self {
        Self { slot_source }
    }

    pub fn some_function(&self) {
        let slot = self.slot_source.get_slot();
        println!("Slot: {}", slot);
        // Additional functionality
    }
}
```

Here's how you might use these structs and enums:

```rs
fn main() {
    let slot_subscriber = SlotSource::SlotSubscriber;
    let order_subscriber = SlotSource::OrderSubscriber;

    let foo1 = FooSubscriber::new(slot_subscriber);
    let foo2 = FooSubscriber::new(order_subscriber);

    foo1.some_function();
    foo2.some_function();
}
```

## Comparing Enums and Traits

Using enums for polymorphism can be beneficial when the set of possible types is known and fixed. Enums provide a simpler and more explicit way to handle multiple types, and they can be more efficient since the variants are all part of the same type.

However, traits offer more flexibility and extensibility. With traits, you can easily add new implementations without modifying existing code. Traits also allow for more dynamic behavior and can be used with generics to create highly reusable code.

In summary, both traits and enums have their place in Rust. Understanding the strengths and limitations of each approach will help you choose the right tool for your specific use case.


## Benchmarking

To compare the performance of trait-based(dynamic dispatch one and static dispatch one) and enum-based polymorphism in Rust, I conducted a series of benchmarks. 

### Trait-based(dynamic dispatch)

```rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

trait SlotSourceTrait {
    fn get_slot(&self) -> u64;
}

struct SlotSubscriber;

impl SlotSourceTrait for SlotSubscriber {
    fn get_slot(&self) -> u64 {
        1
    }
}

struct OrderSubscriber;

impl SlotSourceTrait for OrderSubscriber {
    fn get_slot(&self) -> u64 {
        2
    }
}

struct FooSubscriberTrait {
    slot_source: Box<dyn SlotSourceTrait>,
}

impl FooSubscriberTrait {
    pub fn new(slot_source: Box<dyn SlotSourceTrait>) -> Self {
        Self { slot_source }
    }

    pub fn some_function(&self) {
        let slot = self.slot_source.get_slot();
        black_box(slot); // Use black_box to prevent optimizations
    }
}

fn benchmark_traits_dynamic(c: &mut Criterion) {
    let slot_subscriber = SlotSubscriber;
    let order_subscriber = OrderSubscriber;

    let foo1 = FooSubscriberTrait::new(Box::new(slot_subscriber));
    let foo2 = FooSubscriberTrait::new(Box::new(order_subscriber));

    c.bench_function("traits_slot_subscriber", |b| {
        b.iter(|| foo1.some_function())
    });

    c.bench_function("traits_order_subscriber", |b| {
        b.iter(|| foo2.some_function())
    });
}

criterion_group!(benches, benchmark_traits_dynamic);
criterion_main!(benches);
```

### Trait-based(static dispatch)

```rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

trait SlotSourceTrait {
    fn get_slot(&self) -> u64;
}

struct SlotSubscriber;

impl SlotSourceTrait for SlotSubscriber {
    fn get_slot(&self) -> u64 {
        1
    }
}

struct OrderSubscriber;

impl SlotSourceTrait for OrderSubscriber {
    fn get_slot(&self) -> u64 {
        2
    }
}

struct FooSubscriberTrait<T: SlotSourceTrait> {
    slot_source: T,
}

impl<T: SlotSourceTrait> FooSubscriberTrait<T> {
    pub fn new(slot_source: T) -> Self {
        Self { slot_source }
    }

    pub fn some_function(&self) {
        let slot = self.slot_source.get_slot();
        black_box(slot); // Use black_box to prevent optimizations
    }
}

fn benchmark_traits(c: &mut Criterion) {
    let slot_subscriber = SlotSubscriber;
    let order_subscriber = OrderSubscriber;

    let foo1 = FooSubscriberTrait::new(slot_subscriber);
    let foo2 = FooSubscriberTrait::new(order_subscriber);

    c.bench_function("traits_slot_subscriber", |b| {
        b.iter(|| foo1.some_function())
    });

    c.bench_function("traits_order_subscriber", |b| {
        b.iter(|| foo2.some_function())
    });
}

criterion_group!(benches, benchmark_traits);
criterion_main!(benches);
```

### Enum-based

```rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

enum SlotSource {
    SlotSubscriber,
    OrderSubscriber,
}

impl SlotSource {
    fn get_slot(&self) -> u64 {
        match self {
            SlotSource::SlotSubscriber => 1,
            SlotSource::OrderSubscriber => 2,
        }
    }
}

struct FooSubscriber {
    slot_source: SlotSource,
}

impl FooSubscriber {
    pub fn new(slot_source: SlotSource) -> Self {
        Self { slot_source }
    }

    pub fn some_function(&self) {
        let slot = self.slot_source.get_slot();
        black_box(slot); // Use black_box to prevent optimizations
    }
}

fn benchmark_enums(c: &mut Criterion) {
    let slot_subscriber = SlotSource::SlotSubscriber;
    let order_subscriber = SlotSource::OrderSubscriber;

    let foo1 = FooSubscriber::new(slot_subscriber);
    let foo2 = FooSubscriber::new(order_subscriber);

    c.bench_function("enums_slot_subscriber", |b| b.iter(|| foo1.some_function()));

    c.bench_function("enums_order_subscriber", |b| {
        b.iter(|| foo2.some_function())
    });
}

criterion_group!(benches, benchmark_enums);
criterion_main!(benches);
```

Here are the results:

### Trait-based(dynamic dispatch)

Performance regressed significantly compared to the static dispatch, indicating the overhead associated with dynamic dispatch.

```bash
Running benches/dynamic.rs (target/release/deps/dynamic-3303b3f3ed1e59d0)
Gnuplot not found, using plotters backend
traits_slot_subscriber  time:   [925.68 ps 926.64 ps 927.76 ps]
                        change: [+407.06% +407.74% +408.41%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 13 outliers among 100 measurements (13.00%)
  13 (13.00%) high severe

traits_order_subscriber time:   [912.04 ps 912.54 ps 913.51 ps]
                        change: [+397.81% +398.31% +398.86%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 6 outliers among 100 measurements (6.00%)
  3 (3.00%) high mild
  3 (3.00%) high severe
```

### Trait-based(static dispatch)

Performance improved dramatically compared to dynamic dispatch, highlighting the benefits of static dispatch when using traits.

```bash
Running benches/static.rs (target/release/deps/static-3987288cf744c972)
Gnuplot not found, using plotters backend
traits_slot_subscriber  time:   [182.78 ps 182.86 ps 182.99 ps]
                        change: [-80.276% -80.257% -80.239%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 7 outliers among 100 measurements (7.00%)
  1 (1.00%) high mild
  6 (6.00%) high severe

traits_order_subscriber time:   [182.90 ps 183.21 ps 183.59 ps]
                        change: [-79.945% -79.873% -79.790%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 8 outliers among 100 measurements (8.00%)
  4 (4.00%) high mild
  4 (4.00%) high severe
```

### Enum-based

Performance remained stable with no significant change detected. Enums provide a more consistent and efficient solution compared to dynamic dispatch traits.

```bash
Running benches/enum.rs (target/release/deps/enum-5b9a9ddb913d92ca)
Gnuplot not found, using plotters backend
enums_slot_subscriber   time:   [182.41 ps 182.73 ps 183.24 ps]
                        change: [-0.1698% +0.6072% +1.6609%] (p = 0.21 > 0.05)
                        No change in performance detected.
Found 18 outliers among 100 measurements (18.00%)
  8 (8.00%) high mild
  10 (10.00%) high severe

enums_order_subscriber  time:   [182.73 ps 182.81 ps 182.90 ps]
                        change: [+0.2471% +0.3127% +0.3715%] (p = 0.00 < 0.05)
                        Change within noise threshold.
Found 13 outliers among 100 measurements (13.00%)
  1 (1.00%) low mild
  4 (4.00%) high mild
  8 (8.00%) high severe
```

## Conclusion

In this blog post, we've explored two different approaches to achieving polymorphism in Rust: using traits and using enums. We started with a familiar example in JavaScript and translated it into Rust using both traits and enums. We then set up benchmarks to compare the performance of these implementations.

Thank you for reading! I hope this exploration of polymorphism in Rust has been informative and helpful.

## Resources
- https://www.mattkennedy.io/blog/rust_polymorphism/
