# Data Structures: `struct`

**Stage**: 0

**Author**: Eddy Wilson (@eddyw)

## Introduction

Structs are class-like constructors that can only define public/private/accessor class fields (no methods or statics) and have a frozen prototype. Structs have an implicit constructor function that cannot be overwritten. An instance of a struct can be created using the `new` operator or simply calling the constructor as a function. The constructor accepts a single optional argument - an object-like argument - which works similarly to calling `Object.assign(this, argument)` but it only assigns to properties (fields) that have previously been defined in the struct:

```csharp
struct Point {
	X = 0;
	Y;
}
const pt1 = Point();      // Point {X: 0, Y: undefined}
const pt2 = Point({Y:1}); // Point {X: 0, Y: 1}
const pt3 = Point({N:9}); // Point {X: 0, Y: undefined}
const pt4 = Point(pt2);   // Point {X: 0, Y: 1}

Object.isSealed(pt1);             // true
Object.isFrozen(Point);           // true
Object.isFrozen(Point.prototype); // true
```

> Notice that in `pt3`, the constructor is called with an object containing the `N` property. Since the property `N` isn't defined in `Point` public fields, it's ignored.

Struct fields share the same semantics as classes for fields' declarations. For example:

```csharp
struct Point {
	X = 10;
	Y = this.X + 20;
}
const pt1 = Point(); // Point {X: 10, Y: 30}
```

This example roughly desugars to:

```ts
class Point {
	X; // Before constructor(partial) is called
	Y; // Before constructor(partial) is called

	// Prevent extensions but property configuration is still allowed at this point
	#_ = void (this.constructor === Point && Object.preventExtensions(this));

	// Before constructor(partial) is called
	// Fields are initialized here (property configuration can be changed by using decorators here)
	X = 10;
	Y = this.X + 20;

	constructor(partial) {
		// Instances are sealed,
		// Unless it was extended by another struct or class
		if (new.target === Point) Object.seal(this);
		if (partial == null) return;

		"X" in partial && (this.X = partial.X);
		"Y" in partial && (this.Y = partial.Y);
	}
}
Object.freeze(Point);
Object.freeze(Point.prototype);
```

Struct fields are first defined and initialized with a value of `undefined`. The instance then becomes non-extensible. Afterwards, the actual configuration and initialization of the fields happens. Structs should have a predictable set of fields. Meaning that any instantiation should always return an object with the same shape:

```csharp
struct Point {
	X = 10;
	Y = (this[Math.random()] = 20);
}
const pt1 = Point(); // TypeError: Cannot add property 0.19108911054157773, object is not extensible

struct PointWithStatic {
	X = 0;
	Y = (PointWithStatic.OhNo = 10);
}
const pt2 = PointWithStatic(); // TypeError: Cannot add property OhNo, object is not extensible
```

Computed properties are allowed in the same way they're in classes:

```csharp
struct Point {
	[Math.random()] = "ok";
	[Symbol()] = "ok";
}
const pt1 = Point(); // Point {"0.17307905910685828": "ok", [Symbol()]: "ok"}
const pt2 = Point(); // Point {"0.17307905910685828": "ok", [Symbol()]: "ok"}
const pt3 = Point(); // Point {"0.17307905910685828": "ok", [Symbol()]: "ok"}
```

Instances of struct are always sealed. This ensures that instances always have the same object shape and only their values are assignable but non-configurable:

```csharp
struct Point {
	X = 0;
	Y = 0;
}
const pt1 = Point();
pt1.Boo = 123; // TypeError: Cannot add property Boo, object is not extensible
Object.defineProperty(pt1, "X", {get() {}}); // TypeError: Cannot redefine property: X
pt1.X = 1234; // OK
pt1.Y = 1234; // OK
```

Structs and classes can extend other structs. However, structs cannot extend classes:

```csharp
struct Point {
	X = 0;
	Y = 0;
}
class MyPin extends Point {
	getLocation() {
		return [this.X, this.Y];
	}
}
const pt1 = Point({X:1});
const pin1 = new MyPin(pt1);  // MyPin {X: 1, Y: 0}
const pin2 = new MyPin(pin1); // MyPin {X: 1, Y: 0}
const pt2 = Point(pin2);      // Point {X: 1, Y: 0}

Object.isSealed(pt1);  // true
Object.isSealed(pin1); // false
Object.isSealed(pin2); // false
Object.isSealed(pt2);  // true

pin1.getLocation(); // [1, 0]
```

# Motivation

Provide a simple and ergonomic way to create plain-object data structures that can be guaranteed to always be same-shape objects with a familiar syntax (class-like). Because their prototype and own properties (no-statics) are frozen, engines can optimize ahead of time lowering the chances for de-optimizations.

Often - in Domain-Driven design architectures for example - we create tons of instances or simply plain-object which easily de-optimize (specifically talking about V8) functions for difference reasons, one of the most common ones is when object shape is different on every invocation. A function that expects an object becomes eventually megamorphic (see [V8 function optimization](https://erdem.pl/2019/08/v-8-function-optimization))

This is often the case when data comes from a source such as a repository which fetches from a database. Such as this case (simplified):

```ts
const user1 = await users.findOne({_id: 1}); // {_id:1, username:"bob"}
const user2 = await users.findOne({_id: 2}); // {_id:2, nickname:"Alice", username:"alice"}
const user3 = await users.findOne({_id: 3}); // {_id:3, username:"john", nickname:"John"}

const User1 = MapToUserDomain.from(user1); // Monomorphic
const User2 = MapToUserDomain.from(user1); // Polymorphic
const User3 = MapToUserDomain.from(user3); // Megamorphic (de-optimizes, worst case)
```

The method `.from` de-optimizes eventually because it doesn't receive a single object with the same shape, so the function's inline-cache (IC) no longer remembers object shapes so accessing properties of the passed argument becomes slow (de-opt). If the function is called often, then the only solution really is just to reduce the number of object shapes to one in order to improve performance.

By using structs, engines could optimize the creation of instances by keeping an immutable object shape which in the best case scenario could be created at parse time (if no dynamic computed properties).

The above problem could potentially be solved in user land with some extra effort. For instance:

```ts
class User {
	_id;
	username;
	nickname;
	deleted;
	constructor(partial) {
		Object.seal(this);
		Object.assign(this, partial ?? {});
	}
}
```

But how do you know if this is performant? does the engine really optimizes it. You run some performance benchmarks, and you realize that it's twice as slow as just using plain objects. Surely, other functions _may_ not de-optimize anymore but the instantiation is slow. This is a probable more performant version of the above:

```ts
const cachedKeys = ["_id", "deleted", "username", "nickname"];
class User {
	_id;
	deleted;
	username;
	nickname;
	constructor(partial) {
		if (new.target === User) Object.seal(this);
		if (partial == null) return;

		for (const key of cachedKeys) {
			key in partial && (this[key] = partial[key]);
		}
	}
}
```

Sealing the instance is slow, you may decide to remove it since now we have a list of cached keys. Surprisingly enough, you find out later than removing the `Object.seal` did cause object shape reconfiguration by some dark hidden function in your codebase such as this (simplified just to illustrate the problem):

```ts
function unDeleteUser(usr) {
	usr.deleted = undefined;
	delete usr.deleted; // Object is reconfigured
	// { _id, deleted, username, nickname } to
	// { _id, username, nickname }
}
```

## User-land vs engines vs plain-objects

The primary motivation for this proposal is not much about not being able to implement it in user-lang (you can) but rather open for built-in optimizations that can be done by engines. An ergonomic, simple syntax, familiar, and not really require any more lines of code or effort than you would use to define plain objects. For instance, (not that it's recommended but) you could do this:

```csharp
const todo = {
	title: "",
	description: "",
	done: false,
}
const inlineTodo = struct inlineTodo {
	title = "";
	description = "";
	done = false;
}()
```

Ideally, with an extra line of code:

```csharp
struct Todo {
	title = "";
	description = "";
	done = false;
}
const todo = Todo(); // +1 LoC
```

## An alternative for "object literal decorators" proposal

There is a possible idea to introduce decorators to "object literals" in the future (see [Proposal decorators / Extensions](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md)). A secondary motivation for this proposal is to be an alternative to "object literal decorators" which - in my personal opinion - do not really seem to provide a valid use case that would justify their implementation in the language. This is an example of a decorated object literal:

```ts
const X = {
	@decB a: v1,
	@decC b: v2,
}
```

Since structs follow the same semantics as classes for fields declarations, the above could be written as (one time instance):

```csharp
const X = struct X {
	@decB a = v1;
	@decC b = v2;
}();
```

More complex structs could be created using private fields, auto-accessors, and decorators (see [Proposal decorators / #New Class Elements](https://github.com/tc39/proposal-decorators#new-class-elements)):

```csharp
struct ComplexStruct {
	#defaultString = "";
	#defaultNumber = 0;
	@TypeString accessor x = this.#defaultString;
	@TypeNumber accessor y = this.#defaultNumber;
}
```

## Possibility to improve DX and ergonomics with type systems (TypeScript / Flow)

This is not the primary use-case but a consequence of bringing structs to the language. For example, in pseudo-TypeScript, structs could be seen (similarly to classes) as interfaces with a default implementation:

```csharp
struct GeoLocation {
	Lat: number = 0;
	Lon: number = 0;
}
const geo: GeoLocation = GeoLocation();
```

Rather than:

```csharp
interface GeoLocationProps {
	Lat: number;
	Lon: number;
}
class GeoLocation implements GeoLocationProps {
	Lat = 0;
	Lon = 0;
	constructor(partial?: Partial<GeoLocationProps>) {
		/* obscure implementation here */
	}
}
```

# Syntax

This proposal uses `struct` keyword because it's the keyword that's most commonly used to define similar data structures in other programing languages such as `Go` and `C#`. Up for discussion.

# Similar or Related Proposals

- [JavaScript Structs: Fixed Layout Objects](https://github.com/tc39/proposal-structs) - the goals are quite different. It shares some similarities to "Plain Structs" but they're different in how the instances are created and require a constructor.

- [Enum Proposal](https://github.com/Jack-Works/proposal-enum) - enums could technically be defined as structs with the extra property of being auto-incremented (see [EXTENSIONS.md](./EXTENSIONS.md))

- [Decorators / Object literal and property decorators and annotations](https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md) - the "object literal decorators" idea inspired this proposal. Structs share almost all semantics of classes - with a few differences - so decorators can be applied to structs' fields in the same way as they can in classes.
