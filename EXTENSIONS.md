# Future Work (Maybe out-of-scope)

The following are merely ideas which may even be covered by other proposals. Listing them here just for documentation or revisiting them later.

## constant structs

They're defined using `struct const` and are equivalent to a "singleton" instance:

```csharp
struct const Colors {
	RED = 0xff0000;
	GREEN = 0x00ff00;
	BLUE = 0x0000ff;
}
// "desugars" to IISFE (<- I just made this up: Immediately Invoked Struct Frozen Expression):
const Colors = Object.freeze(struct Colors {
	RED = 0xff0000;
	GREEN = 0x00ff00;
	BLUE = 0x0000ff;
}())
```

The `constructor` can escape during instantiation, so it should no longer be callable after the first instance is created:

```csharp
let leak;
struct const Colors {
	that = (leak = this.constructor, 0);
}
leak.name; // Colors
leak(); // TypeError: const struct constructor is not callable
```

## enums

Similarly as `struct const`; the only difference is that the values can be numeric. More precisely, the initial values are implicitly numeric and are auto-incremented. Enums have the following properties:

- A private method named `#_IOTA` which has the signature `(initial?: number, fieldIndex: number): any`
- Any numeric initialized field is assigned to `this.#_IOTA(initial, index)`
- Any non initialized field is implicitly assigned to `this.#_IOTA(undefined, index)`
  - The default behavior is to increment the last `initial` value

```csharp
struct enum Colors {
	// #_IOTA(initial, fieldIndex): any
	RED = 4;  // = this.#_IOTA(4, 0)         // = 4
	GREEN;    // = this.#_IOTA(undefined, 1) // = 5
	BLUE;     // = this.#_IOTA(undefined, 2) // = 6
	PINK = 0; // = this.#_IOTA(0, 3)         // = 0
	LIME;     // = this.#_IOTA(undefined, 4) // = 1
}
```

An example using bitwise flags:

```ts
struct enum Status {
  #_IOTA = iota();
  Unknown;    // = this.#_IOTA(undefined, 0) // 0
  New;        // = this.#_IOTA(undefined, 1) // 1 << 0
  Dirty;      // = this.#_IOTA(undefined, 2) // 1 << 1
  Saved;      // = this.#_IOTA(undefined, 3) // 1 << 2
}
function iota {
  let lastIndex = 0;
  let lastValue = 0;
  return (initial, index) => {
    if (initial !== undefined) throw Error("Bitwise enums cannot have initializer");
    if (index === 0) return 0;

    return 1 << (index - 1);
  }
}
```
