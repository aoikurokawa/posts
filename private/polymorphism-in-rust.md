#### Introduction
One day, I tried to translate following JS code into Rust. 
This is the interface is called `SlotSource` that has `getSlot` method. 
```js
export interface SlotSource {
	getSlot(): number;
}
```

In addition, there are some classes implements `SlotSource`, `SlotSubscriber` and `OrderSubscriber`.

```js
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

One class has `slotSource` property being able to get slot number. 
```js
export class FooSubscriber {
	slotSource: SlotSource;

	constructor(slotSource: SlotSource) {
		this.slotSource = slotSource;
	}

	public some_function(): void {
		let slot = this.slotSource.getSlot();
		...
	}
}
```

If we want to convert following code into Rust, we can write something like this.
Normally, `interface` in JS can be translated into `trait` in Rust. 
```rs
trait SlotSource {
	fn get_slot(&self) -> u64;
}
```

Define each struct `SlotSubscriber` and `OrderSubscriber`, and then implements `SlotSource` for both structs.

```rs
struct SlotSubscriber {}

impl SlotSource for SlotSubscriber {
	fn get_slot(&self) -> u64 {
		...
	}
}

struct OrderSubscriber {}

impl SlotSource for OrderSubscriber {
		fn get_slot(&self) -> u64 {
			...
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
		...
	}
}
```



#### Resources
- https://www.mattkennedy.io/blog/rust_polymorphism/
