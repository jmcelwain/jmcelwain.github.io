+++
tags = ["rust"]
categories = ["code"]
date = "2018-12-6
title = "Learning Rust"
+++

## Learning Rust as an Application Developer

I don't have a CS degree, and was never forced to sit through an operating systems class. I feel pretty comfortable reading C, and have used Linux as my daily driver for years, so I'm not a total novice when it comes to systems, but it's definitely outside of my development experience, which has mainly been with Java and to a lesser extend JavaScript.

Rust has been attractive to me for a long time. I love static typing, and am most comfortable writing an immutable data first style of programming, and Rust seems to sit at the sweet spot between imperative and functional programming. Iterator pipelines are how I write Java, and the more powerful type system in rust feels like a breath of fresh air.

At the same time, the Rust memory model is a significant divergence from what I'm used to. Writing Java, particularly using an IoC framework like Spring has lead me to think in terms of object graphs. I'm not an architecture astronaut who loves to shove OO design patterns into every situation, but I do appreciate writing Java in a clean, idiomatic, "springful" way.

When I started writing Rust, the idea of ownership was difficult for me to get passed. Writing in an OO style, a graph of dependencies can be very complex. While ownership is still an important concept in Java, it's not because of the memory model. A dependency graph in Java is perfectly capable of having cycles, and dependencies are usually expressed in terms of good behavior rather than lifetimes of components. When most everything is a singleton or scoped to a request, it's a lot more tractable to think about what owns what.

It's taken me a while, but I've finally become comfortable thinking of things in terms of the Rust memory model. Although the Rust docs are great, I have to give credit to *Programming Rust: Fast, Safe Systems Development*, which is an exemplary example of programming texts written at an intermediate level. Where before I was constantly fighting the compiler by adding `&` or `&mut` until things compiled, I finally feel comfortable expressing things in terms of ownership rather than singletons and services.