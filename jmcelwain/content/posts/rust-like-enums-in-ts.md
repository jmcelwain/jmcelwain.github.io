+++
tags = ["rust", "typescript"]
date = "2019-02-05"
title = "Rust-like Error and Option Enums in TypeScript"
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

However, this isn't very useful. While the `value` field in `Some<T>` is public, we really shouldn't encourage users to directly access these values. We want to force them to handle both variants. As such, we'll need an interface for `match`:
```typescript
interface Matchable<T> {
    match<E>(fns: { some: (t: T) => E, none: () => E }): E;
}
```

This interface defines a single method which takes an object with two methods representing each of our variants, one of which will be called depending on the state of the option and produces any value defined by the type parameter `E`.

Unfortunately, actually implementing this interface poses a few problems. First, we can try having each variant implement the interface directly:
```typescript
interface Some<T> extends Matchable<T> {
    // ...
}

interface None extends Matchable<never> {
    // ...
}

class Some<T> implements Some<T>{
    // ...
}

class None implements None, Matchable<never> {
    // ...
}
```

Because the `None` variant should never produce a value of type `T`, `never` seems like an appropriate type. However, when we actually try to call this method, we recieve an error:
