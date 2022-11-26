---
title: "The inner workings of ZigSelf's actor model"
date: 2022-11-26T21:00:00Z
tags:
  - self
  - zigself
  - actor-model
  - technical
categories:
  - zigself
type: post
---

Hi there. I wanted to do a series of deep dives about various parts of ZigSelf
in order to give others a more thorough explanation of how everything works, and
also as a braindump in order to verify my thinking. The first post I wanted to
make is regarding ZigSelf's implementation of the actor model, as it's the
feature I believe is one of the most interesting aspects of the language.

The post starts off with a high-level overview and then gets more technical. The
finer details of the scheduler aren't really as important as the concept behind
it, which I've hopefully made as clear as possible.

# Background

[ZigSelf](https://github.com/sin-ack/zigself) is an implementation of the [Self
programming language](https://selflanguage.org/) in [Zig](https://ziglang.org).
It is a pure prototype-based object programming language where the main method
of executing code is by sending a message to objects. The virtual machine part
is written in Zig while the standard library is written in ZigSelf code.

Early in 2022, I was looking at ways to get a concurrency model in ZigSelf. I'm
a big fan of structured concurrency, and while the original Self did have its
own concurrency model, it was a completely unprotected system - Self processes
are preemptively multitasked, and there is no protection between the different
processes regarding memory isolation, which meant a fallback to traditional
locking-based multitasking. I felt that we could do better, as I already had a
managed VM that is able to intercept each object access. While I was looking
into how other programming languages did this, I came across Erlang, and it felt
almost perfect.

# How Erlang Does It

[Erlang](https://www.erlang.org/) is a functional programming language designed
to be highly concurrent. The whole system runs on what it calls the _actor
model_, where many isolated processes concurrently execute and communicate by
message sending. There are two important points to the design here:

1. All Erlang values are immutable (which kind of comes with the functional
   territory). This makes it rather cheap to send messages to other processes,
   because there is no copying involved beyond compiler optimizations; neither
   the sender and receiver can change the object anyway.
2. Erlang code is written as modules, compiled ahead of time and then run later.
   This is important, because the standard functions one uses cannot change
   while the program is running (beyond code replacement, but that's rare),
   which also removes any worries of someone messing around with the functions
   being used.

Both of these points make Erlang a really good candidate for the actor model.
They also make ZigSelf a really bad candidate for it.

## ZigSelf is a bad language for the actor model

The main issue stems from those two points I touched upon. The fact that ZigSelf
is an object oriented programming language, and the fact that it is intended to
be used as a live system and not a "compile and deploy" type of system means
that actors now have many more hazards that have to be avoided. Let's revisit
those two points:

1. In ZigSelf, objects can have assignable slots which can be modified at will.
   This means that sending an object as a message must now copy at least the
   writable parts of the object graph, because both the sender and receiver can
   now affect each other.
2. I intend for ZigSelf to be used as a fully live visual programming
   environment. Once one introduces the actor model to this equation, all bets
   are off for code that needs to access well-known things; the global object
   hierarchy can be changed at any time, and all running actors have to cope
   with this (this is not a bug -- everything in the system should be
   modifiable!).

## Why I think ZigSelf should have the actor model

Even though these issues made it look like the actor model was very wrong for
ZigSelf, I was still adamant that it was _the one_. I had several reasons:

1. In ZigSelf, message passing is the main "calling convention" - objects
   respond to messages sent to them, similar to languages like Smalltalk or
   Ruby. The actor model works in almost exactly the same way, except the
   responses are asynchronous and disconnected.
2. In Erlang, processes are isolated bits of running code with their own memory.
   Turns out objects in ZigSelf are basically the same thing ([objects are poor
   man's closures](https://stackoverflow.com/a/11421598))! So if we use an
   object as our _actor context_ and send a message to that object, we get
   something akin to an Erlang process.
3. While the original Self programming environment's model was much closer in
   effect to OS threads than something like the actor model, it still was a
   process-based model, telling me that if the right balance could be found,
   this model is feasible.

So, how can we make the actor model work with ZigSelf? We're going to have to
tackle the various problems that we've talked about so far.

# Glossary

Let's start with some terminology, because the explanation below can get verbose
without some terms. Feel free to skip this part and reference it whenever an
unknown term is used.

- **Actor.** An actor is a single-threaded piece of code that can execute
  concurrently with other actors.
- **Actor object.** The actor object is the representation of a single actor
  within the object heap.
- **Actor context.** The actor context is the object that is returned in
  response to the spawn message. The actor context can be used to store the
  state of an actor.
- **Actor mode.** In ZigSelf, _actor mode_ is entered once the `_Genesis:`
  message is sent to an object. This creates the _genesis actor_, which acts as
  the scheduler for regular actors.
- **Actor spawner.** The process that spawned the new actor via `_ActorSpawn:`.
  Keep in mind that `_Genesis:` also spawns an actor but is considered special.
- **Global actor.** The global actor is the one that code that's not in actor
  mode executes in. The global actor basically just behaves like a
  single-threaded language runtime such as JavaScript.
- **Genesis actor.** The genesis actor is the only special actor in actor mode.
  It is responsible for scheduling actors and handling the various states that
  an actor can enter.
- **Regular actor.** A regular actor is one that runs user code. It is usually
  application code or the programming environment, and works like an individual
  operating system process. Regular actors are spawned by sending the
  `_ActorSpawn:` message.
- **Spawn message.** The spawn message is the message that is sent to an object
  in order to obtain a new actor context.
- **Entrypoint message.** The entrypoint message is the message sent to the
  actor context object in order to make it start executing. It is the "main
  function" of the actor (and is named `main` in the ZigSelf implementation by
  default, in fact).
- **Blessing.** Blessing is an operation performed on an object graph in order
  to make copy its writable parts for a new actor. This is used during spawning
  and actor messages. Blessing does not copy globally reachable objects.
- **Globally reachable object.** Globally reachable objects are any objects that
  can be reached through the lobby (the root object of ZigSelf). Globally
  reachable objects are read-only during actor mode.
- **Primitive.** A primitive is a message that is defined in the VM. They are
  always prefixed with an underscore (which normal methods cannot be prefixed
  by). They are similar to "built-ins" in other programming languages.
- **Receiver.** Each message has a _receiver_, which is the object that message
  lookup happens on. Within a method activation, `self` refers to the receiver
  of the current activation. Other languages either call this object `self` or
  `this`. For instance, in the message send `object foo: 1 Bar: 2.`, the message
  `foo:Bar:` is sent to the receiver `object`, with the arguments `1` and `2`.

# The Anatomy of an Actor

Before we talk about how ZigSelf implements the actor model, it is important
that I give some information about what an actor looks like in ZigSelf.

The most important parts of an actor is its stacks and its mailbox. There are
various stacks in use by each actor:

- The activation stack, which is used like a regular call stack in other
  programming languages.
- The slot stack, which is used while building objects via literals.
- The argument stack, which is used for arguments while sending messages.
- The saved register stack, which is used for saving registers during calls.

Normally, all of these would be within the same stack, but I found it to be much
nicer to keep them in separate stacks. Additionally, because the stacks are
separate, this makes it much easier to do a precise scan on them during a
garbage collection cycle, and also to do it faster (because items of the same
kind are together, which works better with cache lines).

The mailbox of an actor stores the messages that have been sent to an actor
since its last run. When an actor is resumed, it first answers all messages sent
to it before resuming from its last waiting condition.

In addition to the above, actors also store a reference to their actor context
in order to keep it alive, the _yield reason_ of the actor (if it's suspended),
the file descriptor the actor is currently blocked on (if any), and various
other small bits.

# Entering Actor Mode

In order to enter actor mode, one must first enter the _genesis actor_. This is
done by using the `_Genesis:` primitive. This primitive takes the name of a
message, _blesses_ the current object and sends the message to the new object.

The genesis actor acts as a _scheduler_ for the other actors; when a new actor
is spawned, the genesis actor gets the actor object, and is responsible for
resuming each actor. In addition, the genesis actor also has access to which
file descriptor each actor is blocked on, and can use this information to wait
until a specific event happens, behaving similarly to an event loop. The low
level details of actor states and how the standard library scheduler works is
detailed in [Actor Scheduling in Userspace](#actor-scheduling-in-userspace).

Once the main activation of the genesis actor is exited, the VM exits actor mode
and control returns to the global actor. The ZigSelf VM can enter and exit actor
mode as many times as the user wishes, but there can only be one genesis actor
at a time.

# Creating a New Regular Actor

In Erlang, in order to create a new process, one calls the `spawn` family of
functions (which has different arities depending on what the process creator
wants to happen). This creates the process and then returns the _process ID_
(PID), which the process creator can then use in order to send messages, manage
and otherwise communicate with the process.

ZigSelf works similarly to this. There is a primitive called `_ActorSpawn:`
which allows an actor to send a message to an object in order to spawn a new
actor. This primitive runs through the following steps:

1. Send the message with the given name to the receiver (this is the "spawn
   message"). This message is executed to completion within the actor spawner,
   and suspension of the current actor is not permitted during this time. The
   object that's returned in response to this message becomes the actor context.
   In addition, the spawn message is responsible for calling
   `_ActorSetEntrypoint:`, which tells the `_ActorSpawn:` primitive what message
   to send as the entrypoint message for this actor.
2. The new actor is created.
3. The returned object is "blessed" to be owned by the newly created actor. The
   details of why this is necessary is given in [Actor Memory
   Ownership](#actor-memory-ownership).
4. The entrypoint message that was set during the spawn message is sent to the
   new actor's context.

Once all of these steps are taken, the new actor object is returned. However,
how it is returned depends on the actor spawner:

1. If the actor spawner is the genesis actor, the `Actor` object is the
   response.
2. If the actor spawner is a regular actor, then there are two responses: the
   actor spawner gets a `ActorProxy` which it can use to send messages, and is
   immediately suspended and the `Actor` object is the response of the actor
   spawner's execution for the genesis actor. The genesis actor can then manage
   the `Actor` object as needed and resume the actor spawner.

(Note: ZigSelf currently doesn't allow one to send arguments in an actor spawn
message.)

# Memory Isolation

In Erlang, all process memory is isolated. This means that a process cannot
access memory owned by another process. This completely solves the shared memory
synchronization problem in conventional multi-threading. However, solving this
problem comes with its own peculiarities, namely the fact that the language's
model has to change significantly to adapt to this (which is why Erlang data
structures are immutable).

ZigSelf is an object programming language and mutating objects by sending them
messages is the natural way of using it. As such, one must take extra care when
designing a memory isolation system. ZigSelf employs a few different methods to
ensure that memory is properly isolated between actors.

## Actor memory ownership

Earlier I had mentioned that each actor owns its memory and there is no memory
sharing allowed between actors. This is ensured by marking every newly-created
object with the actor that created it. Reading other actors' memory is prevented
by simply not making it possible to reference that memory in the first place;
getting one's hands on an object owned by another actor is an immediate VM crash
since that would mean that a VM invariant is broken.

However, there is one small wrinkle in this model: spawning of new actors. In
[Creating a New Regular Actor](#creating-a-new-regular-actor), I mentioned that
the object that's returned as the response of the spawn message is "blessed".
But why is this necessary?

The problem here is a variation of the chicken and egg problem: In order for the
new actor to be able to clone memory, it needs to be able to somehow reach the
object it wants to clone. In ZigSelf, nothing is actually truly "global": one
reaches the global objects by going up the lookup chain until the lobby (the
root object) is reached, after which one sends a series of messages to the lobby
in order to find the well-known object (for example, `std vector` is just `std`
being sent to the current activation object, and `vector` being sent to the
response of that message).

In the last sentence I said "current activation object". That is what's referred
to as `self` within a method. The problem becomes obvious after this: if the
actor doesn't have any memory of its own yet, it can't have anything to call
`self`, and so running any non-trivial code at all would be an instant crash.
This is the reason the spawn message is executed within the actor spawner, and
it is also why the "blessing" step is needed -- once the new actor context is
returned from the spawn message, it is then made to belong to the new actor, and
the new actor can now execute code through its own memory.

## Read-only global object hierarchy

Of course, the aforementioned memory ownership works fine for objects owned by
the actor itself, but one can't really write useful code without being able to
reuse code from places like the standard library (you could definitely stuff all
your required definitions within your actor context, but nobody actually does
that, right? :^). This is where we hit another problem: ZigSelf doesn't have a
module system in the traditional sense where code is physically separated. This
is not a bug - it is actually one of the great features of the snapshot-based
virtual machine model. It allows for one to pack everything the program needs
within a single file and carry it elsewhere, and need only the virtual machine
to handle the non-portable parts.

However, this also means that all actors have to share the same global object
hierarchy. This means (_gasp_) shared memory! So how can ZigSelf solve this
problem while keeping things ergonomic?

From my observations working with ZigSelf for the past year, the global object
hierarchy is seldom modified during normal runtime (to the point where it could
be considered completely static). Then it hit me - why not just make it static?
We could freeze the global hierarchy, at least during normal actor operation,
and stop the world during those few moments when someone actually modifies the
global object hierarchy.

This idea gives us the best of both worlds: Keep the system fully live and
malleable, and make it possible for actors to work without significant slowdowns
caused by locking for each global access.

Here's how it works: In addition to the owning actor for each object, the most
significant bit of each object's header word contains a "globally reachable"
bit. This bit is poisonous; objects that are made reachable through a globally
reachable object via any means become marked as globally reachable themselves.
Initially, only the base object traits and the root object are marked as
"globally reachable", and it propagates as the standard library is built during
world creation.

While actor mode is off, globally reachable objects can be modified freely. Once
actor mode is turned on however, all globally reachable objects are put into
read-only mode. There is currently no way to modify the global hierarchy in
actor mode, but I intend to create a primitive that allows this soon.

The addition of the globally reachable bit actually aids actor memory isolation.
Because globally reachable objects are read-only during regular actor operation,
there is no need to copy them; they behave similarly to Erlang's immutable data
structures in this sense. Therefore, the blessing operation skips parts of the
object graph that are globally reachable, which makes things like blessing a
standard library data structure as simple as deep copying the data structure's
contents, without bringing the whole standard library with it.

## Actor message copying

At the start, I mentioned that because of ZigSelf's mutable nature, objects sent
as part of actor messages have to be copied. This is done by just combining the
previous two points: all non-globally reachable objects that are part of a sent
message's object graph are copied and a reference is stored to the copy in the
recipient's mailbox.

When an actor is resumed, it first looks into the mailbox and executes any
messages that it has received. Any invalid messages are handled during the
initial send to the ActorProxy object, and a reference to the method object is
stored on the mailbox, so there is no possibility of a "delayed" method lookup
error. All messages in the mailbox are executed before control returns to the
point where the actor had previously been paused.

This actually works surprisingly well in practice, because most messages that
are sent include standard things like integers, vectors and byte arrays
(strings).

# Actor Scheduling in Userspace

When I was looking into how other programming languages handled their
concurrency, I noticed a distinct separation between languages that delegate
scheduling to userspace and languages that handle scheduling directly within the
virtual machine. For example, when Python introduced asynchronous support in 3.5
it decided that handling of coroutines should be delegated to an event loop
implementation in Python code. I found that to be much more closely aligned with
my own goals, because it has spawned great projects like
[Trio](https://github.com/python-trio/trio) which provide very cool improvements
over Python's default event loop implementation. Comparing this with languages
like Javascript where the virtual machine defines how the event loop is to be
executed without allowing any introspection or replacement made the choice to
put the scheduler in userspace obvious.

In ZigSelf, there are various primitives provided by the virtual machine that
allows one to control the running state of an actor and also to get various bits
of information from it. These primitives will be explored shortly, but first
let's take a look at how the scheduler in ZigSelf's standard library currently
executes processes at a high level.

```smalltalk
(|
    parent* = std traits clonable.

    readyQueue = std vector copy.
    blockedQueue = std vector copy.

    "Given a yielded actor, execute the correct action based on its
     yield reason."
    handleSuspensionOf: actor WithResult: result = (|
        reason.
        yieldReason = std actor yieldReason.
    |
        reason: actor _ActorYieldReason.

        reason = yieldReason dead ifTrue: [
            "There's nothing to do since the actor has died."
            ^ nil.
        ].

        reason = yieldReason yielded ifTrue: [
            "The actor has yielded on its own and is ready to be
             executed again."
            readyQueue append: actor.
            ^ nil.
        ].

        reason = yieldReason actorSpawned ifTrue: [
            "This actor spawned another actor (which is now in result).
             Put this actor and the new actor in the ready queue."
            readyQueue append: result.
            readyQueue append: actor.
            ^ nil.
        ].

        reason = yieldReason blocked ifTrue: [
            "The actor performed a blocking operation. Put it in the
             blocked queue. blockedActor is a helper object that
             stores the FD the actor is blocked on by sending the
             actor it's given the _ActorBlockedFD primitive."
            blockedQueue append: blockedActor copyActor: actor.
            ^ nil.
        ].

        reason = yieldReason runtimeError ifTrue: [
            "The actor received a runtime error."
            handleRuntimeErrorFor: actor.
            ^ nil.
        ].

        raiseError: 'Actor yielded for unknown reason'.
    ).

    schedule = (
        "Start the first actor somehow. This is done in the actual
         implementation via std scheduler startBySending:To:."
        readyQueue append: spawnTheFirstActorSomehow.

        [| :break. :continue |
            "Check if there are any ready processes to execute."
            readyQueue isEmpty not ifTrue: [| firstReadyActor. result. |
                firstReadyActor: readyQueue shift.
                "Resume the actor from where it left off."
                result: firstReadyActor _ActorResume.
                "Once we get here, the actor has executed and
                 suspended again. Figure out why the actor was
                 suspended and add it to the correct queue."
                handleSuspensionOf: firstReadyActor WithResult: result.

                "Continue with scheduling."
                continue value.
            ].

            "There were no ready processes. If there are no blocked
             processes either, then everything has successfully executed
             and we can exit the scheduler."
            blockedQueue isEmpty ifTrue: break.

            "There were blocked processes. Let's wait until at least one
             of them become unblocked."
            waitForBlockedProcesses.
        ] loopBreakContinue.
    ).
|) _Genesis: 'schedule'.
```

I'm omitting the implementation of various functions here, but this is the
overall structure of the ZigSelf scheduler as of writing. The most important
parts here are:

- `_ActorSpawn:` - This is how new actors are created. The initial actor is
  created by giving the scheduler an object to send a message to, and the rest
  is handled by the scheduler as can be seen above.
- `_ActorResume` - This is the most important part of the actor model in
  ZigSelf. It is essentially a [context
  switch](https://en.wikipedia.org/wiki/Context_switch) - it switches from the
  currently executing actor to another one and resumes it from where it had
  previously left off.
- `_ActorYieldReason` - An actor may have paused for various reasons, and this
  is how the scheduler learns why an actor has stopped. This information can
  then be used in order to guide what the scheduler should do with the actor, as
  can be seen above.
- `_ActorBlockedFD` - The file descriptor that the actor is blocked on. Certain
  operations such as reading will block the actor instead of the entire virtual
  machine when actor mode is enabled, and the file descriptor that the actor
  tried to read from will be stored in order to give it to the scheduler.

These four primitives allow us to implement a scheduler in userspace. The choice
to expose these attributes was deliberate, as explained above.

# An Actor's Lifecycle

In the previous section we looked at how the scheduler is implemented in code,
but I think the best way to explore a ZigSelf actor's lifecycle is with a graph.
Below, you can see a state diagram for an actor.

{{< figure src="/images/zigself-actor/state-diagram.png" caption=`State diagram
for the lifecycle of an actor.` >}}

There are currently 5 reasons that a regular actor can yield:

- **Yielded.** This is the simplest reason. The actor can send the `_ActorYield`
  message at any time in order to pause its execution. This is useful for things
  like servers where the actor can yield itself after handling incoming messages
  in order to give other actors a chance to run.
- **Blocked.** The actor tried to perform an operation that would block and is
  now suspended. The scheduler can resume the actor once the operation no longer
  blocks. Keep in mind that this yield reason causes the last instruction that
  was executed to be re-executed once the actor is resumed (blocking primitives
  are atomic, so they can be retried if they caused a block).
- **Runtime error.** The actor received a runtime error due to a programming
  error. Runtime errors are different from regular errors and cannot be caught.
  This also borrows from Erlang's "let it fail" style.
- **Dead.** The actor has successfully executed all of its code and has exited
  successfully. There is nothing else to be done.
- **Actor spawned.** The actor sent the `_ActorSpawn:` message to an object and
  successfully spawned another actor. As explained in [Creating a New Regular
  Actor](#creating-a-new-regular-actor), this pauses the spawning actor and
  returns the actor object to the genesis actor. The spawning actor can then be
  resumed normally.

The diagram above explains what the current scheduler does for each action.

# Actor Messages

A single actor by itself isn't very useful, because an actor can only represent
a single concurrent computation. In order to take advantage of the full
potential of the actor model, we need to have multiple actors, and those actors
need to be able to communicate with each other. This is where message passing
comes in.

In ZigSelf, sending a message to another actor is extremely simple: You just
need the actor proxy object for that actor (which is given to you via
`_ActorSpawn:`), and you send a message to that actor proxy object. All actor
messages are currently asynchronous.

When a message is sent to the actor proxy, the message is looked up on the
target actor's context. If the result of the lookup is not a method, then it's
simply discarded. This allows for stubbing out some messages. Otherwise, a
reference to the method object and all the passed arguments are saved into the
actor's mailbox so that it can handle the messages the next time it's resumed.

# Unanswered Questions

The eventual goal of ZigSelf's actor model is to schedule actors on multiple
threads at once, which is possible due to the model itself preventing shared
memory as much as possible. However, there are still various open questions
that have to be answered before this can be fully realized.

For example, if we execute multiple actors at once, how does the scheduler run?
In the current model, the scheduler is just another actor and takes control when
the currently running actor suspends. However, once we are scheduling multiple
actors at the same time, how can the scheduler handle all of them at once? For
this, I see a couple solutions:

1. Adding onto the genesis actor's special properties, make it execute on all
   cores at once. This means a return to the unsafe concurrency model but only
   for the scheduler.
2. Use an extra thread just for the genesis actor. This is based on the
   hypothesis that the scheduler will have less to do than all the actors and so
   will be able to handle the load, but comes with the problem of having to
   manage both unblocking blocked actors and scheduling ready ones at the same
   time. Additionally, since the scheduler will now have control even when
   actors are running, receiving new actors as a response to `_ActorSpawn:`s may
   pose a problem.

More importantly, there is the issue of concurrent garbage collection. ZigSelf's
object heap implementation currently uses no locks, which works fine because
there is only one thread of execution. However, once multiple threads are
involved, this poses a great problem: because almost all operations will produce
new objects, if the current model is kept, a lot of performance will be
sacrificed by locking for every object creation. The simplest solution here
would be completely separating the heaps for each actor, and this seems to be
the approach employed by Erlang itself; however this might be prohibitively
expensive, so a compromise could be to separate just the eden (the area where
new objects get created) at a reasonable level of granularity (per thread? per
actor?) and then lock only when a tenure from an eden to the shared eden space
happens.

Furthermore, some things such as file descriptors are process-wide and must be
shared. Currently, ZigSelf wraps such resources with a `Managed` object so
sharing can be managed here. Perhaps an implementation of move semantics is
possible, where sending an actor a message with a `Managed` object "moves" that
`Managed` object to the receiving actor.

Finally, there are some cases where sharing memory is either necessary or
greatly beneficial to performance, such as when parallel operations are
performed over a large block of memory (such as parallel rendering). I don't
have ideas on how to handle this just yet, but I think we could allow
compromises here through some other mechanism.

# Future Goals

As of now, the actor model is almost fully integrated into ZigSelf. However,
there are many goals I want to accomplish before it's ready for prime-time.

Firstly, there are many low-hanging fruit regarding performance in actor message
sending, such as copy-on-write. The current message sending implementation is
"the simplest thing that could possibly work", and I intend to do various
optimizations in this area to minimize the impact of message sending.

Moreover, the mailbox is currently implemented as a linked list because I
believed at the time that it was possible that the mailbox wouldn't be emptied
before receiving messages again. In practice this doesn't seem to be the case,
and the mailbox can be implemented as a simple vector, greatly reducing the
required allocations to maintain it (and therefore reduce the overhead).

And finally, I also want to reduce the disconnect between a message and its
response. In Erlang, messages are distinguished by atoms sent as the first
element of the message, and the response is handled in a similar way. I believe
that we can have better abstractions. In particular, I want to have a
promise-like system where one can wait on the response of an asynchronous actor
message without expecting a specific string or atom. Alternatively, some
messages could be made synchronous, where the sending actor suspends itself and
runs the other actor in its place with just that message before regaining
control once the response is calculated. This could be a good alternative to
reduce latency for some messages where the execution time is expected to be
small.

# Conclusion

In this post I went over the state of the actor model implementation in ZigSelf
as of November 2022. There are still many challenges and unanswered questions
before I can fully realize my dream of an actor-based object programming system
that runs on multiple cores, but if it were so easy it wouldn't be fun :^) I
would recommend keeping an eye on the [open
issues](https://github.com/sin-ack/zigself/issues) to track the progress of the
actor model.

I know I said that I would do monthly updates regarding ZigSelf, however I
unfortunately didn't have time to work on ZigSelf during the past few months so
there was nothing to write about. However, December seems promising. Stay tuned!

Oh, also: ZigSelf now has a [Discord server](https://discord.gg/HJ62kw6yvn),
where you can talk to me and others about the language, and ask any
questions/make suggestions about this post. Hoping to see you over there.

# Aside: Why the name "genesis"?

People have asked me why I chose the name "genesis" for the primitive that lets
one enter actor mode. The name is an homage to the original Self programming
environment, which calls the place where new objects are created "eden" (a name
which ZigSelf also uses). The name of the "blessing" operation follows a similar
theme. :^)
