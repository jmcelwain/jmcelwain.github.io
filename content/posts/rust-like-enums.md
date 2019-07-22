+++
tags = ["rust", "kafka", "raft"]
date = "2019-03-08"
title = "Rust-like Enumes in TypeScript"
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

First, we can try having each variant implement the interface directly. However, the type parameter `T` on the interface poses a problem for `None`: if the none or nil variant can never produce a value, it doesn't make sense to be parameterized over `T`. In truth, `None` needs implements `Matchable<never>`, because the user should never be able to get a `T` out of it -- we will always call the "none" handler in the match.

This is problematic. If each variant implements `Matchable` directly, TypeScript produces an error when trying to call `match` on an instance of `Option`:

```typescript
let maybeNumber = Option.of(1)
maybeNumber.match({
    some(num) {
        // ...
    }
    none() {
        // ...
    }
}) // <--- ERROR!
```
```
[ts] Cannot invoke an expression whose type lacks a call signature. Type '(<E>(fns: { some: (t: never) => E; none: () => E; }) => E) | (<E>(fns: { some: (t: number) => E; none: () => E; }) => E)' has no compatible call signatures. [2349]
```

We cannot call `match` because typescript does not know how to call it. The problem is with the method `some` in our `None` variant, which has the concrete signature `some<E>(val: never): E`. A method with a parameter that is `never` can never be called:

```typescript
type MaybeFn<T, E> = ((a: never) => E) | ((a: T) => E)
let maybeThree: MaybeFn<any, number> = (_n: any) => 3

maybeThree("foo")
        // ~~~~~ Argument of type "foo" is not assignable to parameter of type 'never'. [2345]
```

`never` is the "bottom" type, which effectively means that it cannot be represented. We pass a concrete value as `never` and any function returning `never` is expected to diverge, i.e. never return, by throwing an exception or exiting.