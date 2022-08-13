---
title: "ZigSelf Update, August 2022"
date: 2022-08-13T10:00:00+01:00
tags:
  - self
  - zigself
  - update
categories:
  - zigself

type: post
---

Hi there. It's been quite a while since I posted here, mostly due to real life
responsibilities. I'm planning on making monthly updates here from now on so
that I can showcase the new things that are in ZigSelf. For the first update
though, let's play some catch-up.

<!--more-->

# Language

## The start of an actor model, inspired by Erlang

This was something I had intended from the beginning. The original Self VM
relies on what it calls "processes" to operate. Similar to operating system
threads, there is no memory protection between these user processes and a user
must use semaphores. I wanted a more structured approach to my concurrency
model.

When I saw Erlang's actor model, it was an "aha" moment: Self is a language that
naturally uses message passing as its method of communication, and it made
perfect sense to add a message-passing based concurrency model! In this model,
actors would only communicate using messages, and would not be able to access
each other's memory.

One big problem however was that unlike Erlang, ZigSelf is an object-oriented
programming language and is mutable by default. Objects can mutate other
objects, including the global object hierarchy (everything that an object can
reach from lobby, including the standard library). It would be too restrictive
to fully cut off access to the global object hierarchy.

Another important point was that I wanted the scheduling of actors to be fully
in userspace. I believe that putting the scheduler directly in the VM would be
too restrictive. I wanted to have a userspace scheduler, so that it's easily
modifiable and introspectable.

From these ideas, ZigSelf's actor model was born. Fully explaining how it works
would take several pages, and I intend to do that with a dedicated post soon. It
takes the basic idea from Erlang, and makes it possible to safely run multiple
actors at once with read-only global hierarchy access. In the future, I want to
make it possible to schedule actors on multiple OS threads at once, which would
be the ultimate point in ZigSelf's concurrency model; however, there are still
various problems to solve before getting there.

Currently actors can still access each other's memory, but work is already
underway to fix this. I am going to implement actor memory isolation before
going through with writing any code that depends on actors.

## `;` syntax for chaining message sends easily

The `;` symbol allows one to send a message to the result of an expression. This
makes it much easier and better-looking to chain messages in the VM (especially
important considering most mutating methods return `self`):

```zigself
"Before"
((foo bar: 1) baz: 2 Quux: 3) zoink: 4.
(1 + 2) asString printLine.
"After"
foo bar: 1; baz: 2 Quux: 3; zoink: 4.
1 + 2; asString printLine.
```

Not only does it cut down on the amount of parenthesis needed, but it also makes
for very nice chaining syntax. Take this example from
[the standard library](https://github.com/sin-ack/zigself/blob/42e35a7/objects/hashMap.self#L39):

```zigself
copyKey: key Value: value = (clone; key: key; value: value).
```

To supplement the chaining system, assignment messages were made to return
`self` (which is what you're seeing above).

## Unix socket primitives, AKA ZigSelf's first talk with the internet

In early June I implemented some primitives that implement the BSD socket
interface (or at least some of it). This makes it possible for ZigSelf to listen
as a server and make HTTP requests. Additionally, they fully support the actor
model, blocking only the current actor if the VM is in a regular actor :^)

Right now the primitives are _very_ primitive and don't do any proper error
handling, but I plan to change that in the near future. For now, I'll leave you
with this small snippet:

{{< figure src="/images/zigself-202208/host.png" caption=`A basic implementation
of a <code>host</code> program in Self running successfully.` >}}

# Virtual Machine

## The bytecode interpreter

When I was implementing the actor model, it became clear to me that my initial
AST interpreter implementation was not going to be viable. For one, it didn't
support pausing the thread of execution at a certain point, and it would also
require me to unwind and rewind the stack everytime even if it did. Therefore, I
looked into what I needed to do to implement a bytecode interpreter.

The bytecode interpreter could probably be explained in much more detail in
another post. The new interpreter runs a register-based bytecode set, with 3
separate stacks: arguments, slots and saved registers. The argument stack is
used for arguments to a message, the slot stack is used for building objects and
the saved registers stack is used to restore registers after a return operation.

There are currently two bytecode instruction sets in ZigSelf. When the parser is
done parsing a source file, the AST tree is then sent to
[`AstGen`](https://github.com/sin-ack/zigself/blob/42e35a7/src/runtime/AstGen.zig),
which is responsible for generating _ASTcode_. After the ASTcode is generated, a
linear-scan register allocation is performed in order to minimize the amount of
registers used in the VM in
[`CodeGen`](https://github.com/sin-ack/zigself/blob/42e35a7/src/runtime/CodeGen.zig),
and _Lowcode_ is produced. The VM then runs the Lowcode in the global actor and
execution continues as normal. When an actor is paused, the current activation
stack of the actor is paused, and another actor takes its place.

I want to go into much more detail in a follow-up post, but to summarize, the
bytecode VM made it possible to implement the actor model and also gave the VM
a surprising 20% performance boost (despite the two bytecode passes!).

## `HandleArea` to reduce handle allocation churn

[`267358c`](https://github.com/sin-ack/zigself/commit/267358c) introduced a new
structure called `HandleArea` as part of the heap. A `Handle` is a strong
reference to an object in the object heap in Zig code which prevents it from
being garbage collected. This is very useful when some VM code needs objects to
survive a garbage collection. However, previously every single `Handle` would be
heap-allocated due to how `Handle`s work (the contents of the handle must not
move in order for it to work properly).

`HandleArea` significantly reduces memory consumption in this regard by
implementing an arena allocator that can deallocate its chunks. Each chunk is
1MB in size and is self-aligned (so each address is aligned to `0x10000`). This
allows for the `HandleArea` to keep track of how many handles are allocated in a
very simple fashion, and recycle/deallocate `HandleArea` chunks as necessary.
This significantly increased the VM performance.

With the implementation of the bytecode, the `HandleArea` is used much less, but
nevertheless it's still very important in order to prevent unnecessary
allocations.

## Stricter heap allocation

[`e2697ff`](https://github.com/sin-ack/zigself/commit/e2697ff) significantly
changed how allocation within the ZigSelf virtual machine works. Previously, any
heap allocation was allowed to garbage collect. This made it very easy to
allocate from the heap, but came with a problem: any heap allocation garbage
collecting meant that the program could garbage collect in the middle of any
operation, regardless of whether it was safe to do so. To make it possible to
safely perform multiple allocations at once, a method called
`heap.ensureSpaceInEden` was made to garbage collect if there wasn't enough
space, but this was very error-prone as it was quite easy to miscalculate the
amount of space needed and cause a garbage collection anyway.

This has (on more than one occasion) caused confusing and frustrating stale heap
reference bugs, where a heap allocation within the VM (in Zig code) would cause
a garbage collection accidentally and the only way to know what actually
happened was by carefully tracing everything that happened in a debugger.

With the new heap allocation API, the user of the heap allocation is fully
responsible for their memory accounting. A new method called
`heap.getAllocation` can be called with the requested amount of bytes, which
ensures that there's enough space in eden before returning an `AllocationToken`.
The token can then be used to allocate on the heap. If the requested amount of
bytes is exceeded while allocating, the VM immediately crashes, turning
frustrating stale heap reference bugs into easy-to-debug crashes. Additionally,
calling `token.deinit` at the end of the function (via `defer`) ensures that the
caller didn't _under-use_ their requested amount of bytes, thus also preventing
unnecessary garbage collections.

## Lazy allocation of old space

When the `Heap` was first written, memory for all three generations (eden, new
space and old space) were allocated in one go. Because old space is only ever
used in long-running programs (and we don't have any so far), this meant that
16MB of memory was unnecessarily allocated.

[`26f53d0`](https://github.com/sin-ack/zigself/commit/26f53d0) makes it so that
old space is only allocated if it has an object allocated inside it. This
reduces memory usage significantly for short running programs (in my tests short
running and medium-length programs only use about 15-20MB of memory).

## Zig self-hosted compiler support

As of August 13, ZigSelf can now build on both stage1 (the bootstrap compiler)
and stage2 (the self-hosted compiler)! This is a great change, because stage2
brings many improvements, the most notable ones being improvements to
compilation speed and memory usage. ZigSelf will be fully ready to compile with
0.10.0, which will ship with the self-hosted compiler by default.

(Note: The self-hosted compiler support has been tested with `a89c3de66`. I try
to keep up to date with the master branch of Zig as much as possible but it's
possible that compiler changes have broken compilation since writing this post.)

# Standard Library

## Collection API in the standard library

One of the main things that ZigSelf lacked was that it had no good way of
expressing collections of items. The closest thing we had was a linked list,
which was woefully slow for random access (of course). To fix this, I've
implemented a new set of APIs collectively (hehe) called "The Collection API".
It consists of a core trait which defines things that should work on all
collections (checking whether the collection is empty, finding items, etc.),
plus a few mixin traits which define various different traits of the collections
we have. The following concrete collection data structures have been implemented
so far:

- `std array`, a statically-sized array.
- `std vector`, a dynamically-sized array.
- `std hashTable`, a dynamically-resizing open addressed hash table.
- `std hashMap`, a mapping of items built on top of `std hashTable`.
- `std hashSet`, a unique set of items built on top of `std hashTable`.

Additionally, various existing traits like `std string` have been adapted to the
collection API, making it possible to use them just like any other collection.

In the process of making `std vector`, a new object type called `Array` was also
implemented. An `Array` is a statically-sized array, and has various primitives
which allow manipulating it. `std array` is a thin wrapper over an `Array`
object, while `std vector` and `std hashTable` (and its descendants) use it as
the basis of their respective implementations.

You can see the interface
[here](https://github.com/sin-ack/zigself/blob/42e35a7/objects/collection.self).

## `std collector`

One of the main problems with zigSelf syntax (and Self syntax in general) is
that it does not allow for things like array literals, which are present in many
languages but don't exist in Self due to its very simple syntax. To alleviate
this, the `std collector` object has been introduced. `std collector` is
intended to be an easy-to-use interface for constructing various collections:

```zigself
1 & 2 & 3; asVector asString printLine. "[1, 2, 3]"
'hello ' & 'world!'; flattenIntoString printLine. "hello world!"
```

Under the hood, `std collector` has been implemented as a linked list as it does
not need random access and appending an item is O(1). Once a conversion to a
collection has been requested, the linked list is scanned once and the
collection is appended to.

You can see the source code for `std collector`
[here](https://github.com/sin-ack/zigself/blob/42e35a7/objects/collector.self).

---

Thanks for reading! If you're interested in ZigSelf and want to discuss it with
me and others, or want to contribute, you can join the
[Discord server](https://discord.gg/HJ62kw6yvn).

I know that I keep disappearing for months at a time, but I intend to post an
update every month with new things in ZigSelf from now on. Stay tuned!
