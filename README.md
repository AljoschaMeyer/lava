# Lava

A specification and protocol for representing and transmitting values that will change over time in an untrusted decentralized setting. This document presumes that the reader is familiar with the contents of the [magma specification](https://github.com/AljoschaMeyer/magma).

**Status: work in progress. Still sorting out the ideas. The write-up does not yet contain sufficient context for motivating all the design choices.**

Lava is an extension of [magma](https://github.com/AljoschaMeyer/magma) in which one can talk about values which do not exist yet. With magma, the only way of addressing a point of time in the evolution of a changing value is by giving its hash. Lava also allows addressing values by their sequence number together with an identifier for the sequence of values.

Addressing by sequence number can be problematic because there can be many values with the same sequence number, all originating from a common ancestor. In particular if everybody can create new events, addressing by sequence number cannot be assumed to be unique. Lava thus extends events with a signature from a public key signature scheme. Only a holder of the secret key can contribute new values, so if we trust the secret holder to not fork the sequence and create only a single event per sequence number, then these names are unique.

In addition to addressing by sequence number, events can also carry monotonically increasing metadata, which can also be used for addressing.

Even if a sequence is forked, the linking schemes allows efficient detection of this happening, so replication can continue to function by only accepting compatible values and discarding values from incompatible branches. See [bamboo](https://github.com/AljoschaMeyer/bamboo) for more context.

## Precise Encoding

This is horribly unreadable, I am sorry... Math vocabulary is annoying, but I need to make this precise.

Lava extends [magma](https://github.com/AljoschaMeyer/magma), so it requires all of its parameters:

- a monoid `(T, +_T, 0_T)`
- a bijective function `encode_monoid: T -> {0, 1}^*` uniquely mapping monoid values to byte strings
- a naming scheme `(compute_name: {0, 1}^* -> N, verify_name: ({0, 1}^*, N) -> Bool)`
- a `reserved_name` from `N` that can be used to encode that no predecessor exists
- a bijective function `encode_name: N -> {0, 1}^*` uniquely mapping names to byte strings

Let `(M, +_M, 0_M)` be a monoid, and let `<=` be a [total order](https://en.wikipedia.org/wiki/Total_order) on `M`. We call `(M, +_M, 0_M)` a monotonically increasing monoid with respect to `<=` if for all `x, y` in `M` we have `x +_M y <= x`.

An instance of the lava protocol requires as parameters:

- a family `(M)_i` of up to 62 monotonically increasing monoids `(M_i, +_M_i, 0_M_i)` with respect to some `<=_i`
- for each such `(M_i, +_M_i, 0_M_i)` a bijective function `encode_meta_i: M_i -> {0, 1}^*` uniquely mapping monoid values to byte strings

A *raw event* consists of the same data as a magma event together with a metadata value `m_i` for each metadata monoid of `(M)_i`. We denote the set of all the raw events by `R`.

In order to identify sequences, lava also needs:

- a [digital signature scheme](https://en.wikipedia.org/wiki/Digital_signature) of public keys from a set `P`, secret keys from a set `S`, and functions `sign: (S, R) -> {0, 1}^*` and `verify: (P, {0, 1}^*) -> Bool`
- functions `remove_redundancy: {0, 1}^* -> {0, 1}^*` and `reconstruct_redundancy: (P, {0, 1}^*) -> {0, 1}^*` such that for all keypairs `(p, s)` and raw events `r` we have `verify(p, reconstruct_redundancy(p, remove_redundancy(sign(s, r)))) = true`

The purpose of `remove_redundancy` and `reconstruct_redundancy` is to allow efficient transfer of multiple events belonging to the same sequence in case the signature includes data that is redundant among all of them. For example, the signature scheme used in [bamboo](https://github.com/AljoschaMeyer/bamboo) adds the public key and a log id to each entry when signing (in addition to a regular [ed25519](https://ed25519.cr.yp.to/) signature over the entry data and the public key and log id). The lava transport protocol transmits events by removing redundancy at the sender and then reconstructing it at the receiver, so redundant information in the signature scheme can be used without increasing the transmission bandwidth.

A *signed event* consists of a *raw event* together with a signature. It is encoded as follows:

- first encode the magma event
- append the metadata values in order, each encoded via `encode_meta_i`
- append the signature

The *compact encoding* appends not the signature but `remove_redundancy(signature)`.

## Transmission Protocol

The design issues for the transmission protocol can be roughly grouped into three categories:

- Which parts of a value sequence are being requested?
- How are these transmitted, how are forks handled?
- How are multiply request executed concurrently and managed?

### Adressing Sequence Parts

Replication works by requesting the events along a shortest path between two points in the sequence. The starting point is given either by the name of an event (exclusive), or by the reserved name to indicate that the range begins at the very start of the sequence. The end point is given either by the name of another event, or by specifying any of the following: a sequence number difference, a payload size difference, or any number of the totally ordered metadata values. In the first case, the range extends until the end point event (inclusive), or until the first event `e` such that for one of the metadata values, appending the specified value to the accumulated metadata value at the starting event exceeds the value of `e` (exclusive).

This is probably completely incomprehensible, sketching some pseudocode instead...

```rust
struct RequestRange {
    start: Option<EventName>,
    end: RangeEnd,
}

enum RangeEnd {
    Absolute(EventName),
    Relative {
        decreasing: bool,
        sequence_number: Option<u64>,
        size: Option<u64>,
        meta0: Option<Meta0>,
        meta1: Option<Meta1>,
        // ... one field per metadata monoid
    },
}

// A function To be used for computing ranges:
//
// Given an event, return information about the events which immediately
// succeed it, i.e., whose predecessor event is the given one.
fn successor(e: Event) -> SuccessorStatus { magic!() }

enum SuccessorStatus {
    Unique(Event),
    Ambiguous,
    None,
}

fn compute_range(r: RequestRange) -> TODO {
    TODO
}

```




TODO how to do logs
