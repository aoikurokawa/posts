#### Introduction
One day, I translated below JS code into Rust. 
This is the interface that has `getSlot` method. 
```js
export interface SlotSource {
	getSlot(): number;
}
```

In addition, there are some classes implements `SlotSource`.

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

Usually, `interface` in JS can be translated into `trait`. So we can write something like this.
```rs
trait SlotSource {
	fn get_slot() -> u64;
}
```




#### Resources
- https://www.mattkennedy.io/blog/rust_polymorphism/
