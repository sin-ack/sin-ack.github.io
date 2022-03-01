---
title: "ZigSelf: An implementation of Self in Zig"
date: 2022-03-01T10:00:00+01:00
tags:
  - self
  - zigself
categories:
  - zigself

type: post
---

Hello there. I have not posted in a while, and not without good reason. In my
[first article](/posts/a-tour-of-self/) I had mentioned that I was going to make
posts about my own implementation of Self called mySelf. That project never took
off, and my interest shifted to other things. I had largely forgotten about it,
but then LangJam happened.

## LangJam

[LangJam](https://github.com/langjam/langjam) is a programming jam that works
similar to game jams, but instead of developing video games you develop a
programming language based around the theme of the jam. When I heard about it
back in August, I remembered about my interest in implementing Self, and decided
to join the jam. My implementation [went largely nowhere](https://github.com/langjam/jam0001/blob/main/TeamDailyDose),
as it wasn't really interesting from the standpoint of the jam's theme. However,
spending 48 hours to implement the lexer, parser and then a basic interpreter
was pretty fun, and I started wondering again.

## Humble beginnings

After the jam ended, I didn't really go back to Self for a while, as other
projects such as SerenityOS caught my attention again.

When November rolled around, I remembered that
[Advent of Code](https://adventofcode.com/) existed and the idea of solving it
with my own language seemed interesting. With this motivation, on November 6th
2021 [I made the first commit to ZigSelf](https://github.com/sin-ack/zigself/commit/0869adc2d36804ebffe2e94c1e80ed7a98f637e2).

After that commit, I started working on the lexer, parser, AST and then
eventually an interpreter. At first I was working only with primitives, as there
was no standard library to speak of. Eventually however, I started implementing
some basic methods for the primitive objects.

## The GC + Heap project

After I had a basic standard library, I decided to implement some old Advent of
Code problems in ZigSelf. However, I noticed something wrong when I decided to
do this. [2020's Day 1](https://adventofcode.com/2020/day/1) which is supposed
to be one of the easiest took over _20 minutes_ to solve both parts.

The reason for this was my initial implementation of the object model. In my
initial implementation I had used ref-counting and a single Object type that
would hold any type that an object can be (including integers!). This caused an
allocation for every basic operation, including _every single integer
operation_. This was obviously infeasible, and I started planning for a proper
object model.

Of course, it would be disrespectful to write a Self implementation and not use
David Ungar's [generational scavenging garbage
collection](https://people.cs.umass.edu/~emery/classes/cmpsci691s-fall2004/papers/p157-ungar.pdf)
and [the Self memory
model](https://raw.githubusercontent.com/russellallen/self/master/docs/papers/implementation.pdf),
so I decided on implementing a full garbage collector and object model based off
of them. The technical details could be interesting for a later article, but it
was a big undertaking to say the least.

At this point it was late November and I had largely given up on getting my
language ready in time for AoC, so I set out to work on getting my language on a
proper object model.

Due to various real life factors, I had to take a break from the project from
the start of December until January 22nd. After that I finally got my object
heap working, fixed up all the primitives on the system (we don't talk about
`_RemoveSlot:IfFail:`) to use the heap model, and have been working on it ever
since.

## Current state

That brings us to today. As of 2 days ago at the time of writing, ZigSelf now
has a REPL! Feel free to check out the
[instructions](https://github.com/sin-ack/zigself#building-zigself) in order to
play around with it. It's very fun to mess around with the language in the REPL
(when it doesn't crash due to a bug :^).

Other than the REPL, various parts of the language have improved substantially.
There are some basic system calls to open, read and write files, and the
standard library contains objects and methods that one would expect. The object
heap has become stable enough to run proper programs on it.

It's been an incredibly fun project to craft my own programming language and see
it executing. It helps me understand how programming languages actually work
internally. I am personally really happy about how well the object model and the
heap works.

## The future

While I have no plans with a concrete time-frame, I do have some future goals
with this project. The biggest eventual goal is getting a graphical programming
environment like the original Self VM working. With a more modern codebase, I
really believe it could be an intuitive way to perform programming on a live
system.

Other than that, an actor model integration is my goal. I am interested in
Erlang's actor model in particular, and I believe it could be a good fit for the
message passing-based programming paradigm that Self employs. The original Self
VM employs the traditional approach to concurrency with locks and what-have-you,
and it's tricky to get things right; with the actor model I believe much safer
concurrency can be accomplished.

Thanks for reading. You can find the [project here](https://github.com/sin-ack/zigself).
I hope you find it as enjoyable as I do. Now that ZigSelf has evolved enough to
talk about it, I will probably write more articles about various technical
aspects of it very soon. Stay tuned!
