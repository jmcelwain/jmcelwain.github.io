+++
tags = ["rust", "kafka", "raft"]
date = "2019-03-08"
title = "Rust-like Enums in TypeScript"
+++

One of the nicest things about Rust is handling nulls with `Option` and error states with `Result`. Because Rust has good pattern matching, handling these cases becomes trivial as the complier forces you to handle all the cases of an enum in a `match` expression.

Because TypeScript lacks pattern matching, we have to lean on the type system in a different way to get similar semantics. For `Option`, we start with defining the two different states and the union type that represents the `Option`:

```typescript
export interface Some<T> {
    tag: "some"
    value: T
}

export interface None {
    tag: "none"
}

export type Option<T> = Some<T> | None
```

Users should only ever deal with `Option<T>`, which represents a value or null. We'll supply convenience methods later for creating `Some<T>`/`None` instances.

The `tag` field allows typescript to do a primitive form of pattern matching in a `switch` expression. It's also best practice when defining union types so that users can otherwise test for which variant they are recieving.

However, because we want `match` like semantics, we'll define things a little different. First, we have to implement our interfaces for `Some<T>` and `None`:

```typescript
export namespace Option {
    class Some<T> implements Some<T> {
        public tag: "some" = "some"
        public value: T 
    
        public constructor(val: T) {
            this.value = val
        }

        public toString() {
            return `Some{ ${this.value} }`
        }
    }
    
    class None implements None {
        public tag: "none" = "none"
 
        public toString() {
            return `None { }`
        }
    }
```

Using a namespace helps us avoid conflicts with the interfaces defined for our variants. However, these variants should never be constructed directly -- instead we provide the following helpers:

```typescript
export function none(): None {
    return new None()
}

export function of<T>(val: T): Option<T> {
    if (val === null || val === undefined) {
        return Option.none()
    }
    
    return new Some(val)
}
```

However, this isn't very useful. While the `value` field in `Some` is public, we really shouldn't encourage users to directly access these values. We want to force them to handle both variants. As such, we'll need an interface for `match`:

```typescript
interface Matchable<T> {
    match<E>(fns: { some: (t: T) => E, none: () => E }): E;
}
```

This interface defines a single method which takes an object with two methods representing each of our variants, one of which will be called depending on the state of the option and produces any value defined by the type parameter `E`.

Unfortunately, actually implementing this interface poses a few problems.

First, we can try having each variant implement the interface directly:
```typescript
export interface Some<T> extends Matchable<T> {
    tag: "some"
    value: T
}

export interface None extends Matchable<never>{
    tag: "none"
}
``` 


However, the type parameter `T` on the interface poses a problem for `None`: if the none or nil variant can never produce a value, it doesn't make sense to be parameterized over `T`. In truth, `None` needs implements `Matchable<never>`, because the user should never be able to get a `T` out of it -- we will always call the "none" handler in the match.

This is problematic. If each variant implements `Matchable` directly, TypeScript produces an error when trying to call `match` on an instance of `Option`:

```typescript
let maybeNumber = Option.of(1)
maybeNumber.match({
    some(num) {
        // ...
    },
    none() {
        // ...
    }
}) // <--- ERROR!
```

```sh
[ts] Cannot invoke an expression whose type lacks a call signature. Type '(<E>(fns: { some: (t: never) => E; none: () => E; }) => E) | (<E>(fns: { some: (t: number) => E; none: () => E; }) => E)' has no compatible call signatures. [2349]
```

We cannot call `match` because typescript does not know how to call it. More sepcifically, the implementation of `some` for the `None` variant has `never` in its signature, and a method with a parameter that is `never` can never be called.

`never` is the "bottom" type, which means that it cannot be represented. It exists at the level of the type system and cannot have a concrete runtime value. Because a `never` cannot be instantiated, a function returning `never` is expected to diverge, i.e. never return, by throwing an exception or exiting.

Using `never` as a paramater, like we're doing, is a bit more confusing. A function with a `never` parameter cannot be called, because it is impossible to provide a concerete instance of `never` with which to call the function.

An uncallable function might not initially seem very useful, but in our case it ensures that the `None` variant can never produce a value. Because a function with the signature `some(t: never)` can never be called, this enfroces the invariant that the implementation for `None` can only call the `none` method.

However, even though we cannot use the `some` method in the `None` implementation, when calling `match` via a reference to a `Some<T> | None`, TypeScript infers the type  `(Some<T> & Matchable<T>) | (None & Matchable<never>)`, or for purposes of our `match` method. In other words, we *could* be dealing with a `Matchable<never>`.

Here's the solution to our problems:

```typescript
export type Option<T> = (Some<T> | None) & Matchable<T>
```

This exposes another weird thing about `never` -- it is assignable to any type. As the bottom type, `never` is a subtype of every type. I lack the type theory knowledge to fully understand the implications of this, but it presents some interesting behavior:


```typescript
let a: never
let b: number
let c: string

b = c // Type 'string' is not assignable to type 'number'.ts(2322)
b = a // Fine
c = a // Fine
```

This assignment rule also applies to a parameterized interface:
```typescript
interface Matchable<T> {}
let matchableNumber: Matchable<number> = {}
let matchableNever: Matchable<never> = {}

matchableNumber = matchableNever

```

In other words, even though our implementation for `None` implements `Matchable<never>`, because `Matchable<never>` is assignable to `Matchable<T>`, our `None` implementation satisfies the intersection `& Matchable<T>` for `Option`.

Consequently, our `Option` type does what we'd expect:
```typescript
maybeNumber.match({
    some(num) {
        console.log(num) // Prints the number
    },
    none() {
        console.log("none")
    }
})
```

Implementing a `Result` type is an excercise left to the reader, but works very similarly. The obvious downside to this approach in TypeScript is that we are manually implementing pattern matching for each enum type we want to create. We have to manually implement all the utility functions (map, filter, etc.) as well.