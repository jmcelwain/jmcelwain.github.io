+++
tags = ["rust", "kafka", "raft"]
date = "2018-12-07"
title = "A Raft State Machine in Rust"
+++

## Implementing Raft in Rust 

In idiomatic OO Java, a simplistic state machine for Raft could be written something like:

```java
class Raft {
    enum State {
        Follower,
        Candidate,
        Leader
    }

    // Votes we've received from other nodes
    private Map<Long, Boolean> votes;

    // Our current state
    private State state = State.Follower;

    public void apply(VoteMessage message) {
        if (this.state != Candidate) {
            throw new InvalidStateException("Only candidates can receive votes!");
        }

        votes.put(message.id, message.vote);
        if (hasQuorum(votes)) {
            transitionToLeader();
        }
    }

    private void transitionToLeader() {
        if (state != Candidate) {
            throw new InvalidStateException("Cannot transition from follower directly to leader!");
        }
        state = State.Leader;
    }

    // ...
}
```
Such a pattern is largely imperative / procedural, and requires explicit validation at each transition to ensure that the state machine is not in an invalid state. While this pattern can be written in a way that ensures only valid transitions are made, it is error prone and, more importantly, difficult to understand.

Of course, it's possible to extract this validation logic into a generic builder patter. For example, from the [Spring Statemachine](https://projects.spring.io/spring-statemachine/) project:
```java
public StateMachine<States, Events> buildMachine() throws Exception {
    Builder<States, Events> builder = StateMachineBuilder.builder();

    builder.configureStates()
        .withStates()
            .initial(States.STATE1)
            .states(EnumSet.allOf(States.class));

    builder.configureTransitions()
        .withExternal()
            .source(States.STATE1).target(States.STATE2)
            .event(Events.EVENT1)
            .and()
        .withExternal()
            .source(States.STATE2).target(States.STATE1)
            .event(Events.EVENT2);

    return builder.build();
}
```

Although this solves the problem of ensuring the state machine is sound, it still requires lots of boilerplate, and an annotation based meta-programming API that fully embodies Spring "magic".

### Rust

Because Rust has a more expressive type system than Rust, it's possible to encode more safety in our API than relying on runtime validation checks. While there's nothing wrong with runtime assertions representing true invariants of your domain, making these sorts of invalid states impossible via the type system is nice.

### From / Into

Rust provides a pair of traits `From` and `Into` that allow type-safe conversion from one type to another. Because `Into` is a corollary follows from the implementation of `From`, it's only necessary for a struct to implement `From` in order to get a `.into()` method for free.

By encoding the transitions of our state machine via the `From` trait, we are able to represent each state as an independent struct and the type system encodes what transitions are possible. My inspiration for this comes largely from the wonderful blog [Pretty State Machines in Rust](https://hoverbear.org/2016/10/12/rust-state-machine-pattern/) that concludes with a basic sketch of some types for a Raft implementation.


For example, a basic type safe transition from a "follower" state to a "candidate" state can be described as such:
```rust
struct Raft<S> {
    state: S
}

struct Follower {
}

struct Candidate {
}


impl From<Raft<Follower>> for Raft<Candidate> {
    fn from(val: Raft<Follower>) -> Raft<Candidate> {
        Raft {
            state: Candidate { }
        }
    }
}
```

Before we implement `From` for any other states, the only transition that is allowed in our state machine is from `Follower` to `Candidate` in the form `let candidate: Raft<Candidate> = Raft::from(follower)`.

Additionally, Rust's type system allows us to implement specific methods that only apply to a particular variant:
```rust
impl Struct<Candidate> {
    pub fn get_votes(&self) -> (votes, total) {
        self.votes.clone().into_iter()
            .fold((0, 0), |(mut votes, mut total), (u64, u64) | {
                if vote {
                    votes += 1;
                }

                total += 1;
                (votes, total)
            })
    }
}
```

Calling `get_votes` on the leader, for example, yield the following error as `get_votes` is only provided for the implementation `Raft<Candidate>:
```
error: no method named `get_votes` found for type `Raft<Leader>` in the current scope
  --> main.rs:12:15
   |
12 |     leader.get_votes();
   |            ^^^^^^^^^

error: aborting due to previous error
```

By encoding this safety in the type system, we don't need to have unnecessary run time checks to ensure that a method isn't called in an incorrect state.

### Java 

A similar pattern could be implemented in Java using an abstract base class:

```java
abstract class Raft {
    abstract State getState();
}

class Leader extends Raft {

}

class Candidate extends Raft {
    public State getState() {
        return State.Candidate;
    }

    public long getVotes() {
        return votes.values.stream()
            .filter(Function.identity())
            .count;
    }
}

class Follower extends Raft {

}
```

While this pattern gives us much of the type safety of the Rust example, it demonstrates a subtle difference between the memory model of Rust and Java object lifetimes. In Rust, the signature `fn from(val: Raft<Follower>) -> Raft<Candidate>` ensures that the object representing our previous state is fully consumed by the transition function.

In Rust, when a value is moved rather than borrowed from the calling scope, the caller is no longer able to access that value. As such, when we consume a state via `From`, the value we return now represent the single canonical reference to the Raft struct.

In Java, we might declare a method `Candidate from(Follower follower)` that would encode a similar transition. However, because Java lacks the strong ownership semantics of Rust, it's possible for us to commit a number of errors that do not violate the memory safety of Java, but represent serious defects in our program.

Java programmers have to be vigilant not to let objects be published to other threads or scopes. In our example, if a reference has been published, nothing prevents an errant procedure from updating what is now shared state.

While such an error can easily be prevented through viligant code review, the type system in Rust makes this whole process egronomic. More proof that static typing, when done right, is friendly and efficient.