---
title: "AoC 2022 in ZigSelf, day 2"
date: 2022-12-02T06:00:00Z
tags:
  - zigself
  - advent-of-code
categories:
  - zigself
type: post
---

The second day of [Advent of Code](https://adventofcode.com/) was also rather
easy on the question side, like day 1. Rather, the hard part was reading the
question. :^) Here's the solution in
[ZigSelf](https://github.com/sin-ack/zigself).

<!--more-->

You can see the [source code
here](https://github.com/sin-ack/zigself-advent-of-code/blob/2176304c8a5282b419f5480b7a965017986a267f/2022/day2/solve.self),
or in the below section.

<details>
<summary>Source code listing</summary>

```smalltalk
'../../../objects/everything.self' _RunScript.

"Today's theme: inline methods!"
(|
    parent* = std traits singleton.

    "Place input.txt in your cwd"
    input.

    "Hopefully one day we'll get a nice enum system."
    winState = (|
        win = (| parent* = std traits clonable |).
        draw = (| parent* = std traits clonable |).
        loss = (| parent* = std traits clonable |).
    |).

    part1 = (|
        winnerOf: him VS: you = (
            him = 'A' ifTrue: [
                you = 'X' ifTrue: [ ^ winState draw ].
                you = 'Y' ifTrue: [ ^ winState win ].
                you = 'Z' ifTrue: [ ^ winState loss ].
            ].
            him = 'B' ifTrue: [
                you = 'X' ifTrue: [ ^ winState loss ].
                you = 'Y' ifTrue: [ ^ winState draw ].
                you = 'Z' ifTrue: [ ^ winState win ].
            ].
            him = 'C' ifTrue: [
                you = 'X' ifTrue: [ ^ winState win ].
                you = 'Y' ifTrue: [ ^ winState loss ].
                you = 'Z' ifTrue: [ ^ winState draw ].
            ].
            _Error: 'Unknown state'.
        ).

        bonusFor: state = (
            state == winState win ifTrue: [ ^ 6 ].
            state == winState draw ifTrue: [ ^ 3 ].
            state == winState loss ifTrue: [ ^ 0 ].
            _Error: 'Unknown state'.
        ).

        scoreFor: you = (
            you = 'X' ifTrue: [ ^ 1 ].
            you = 'Y' ifTrue: [ ^ 2 ].
            you = 'Z' ifTrue: [ ^ 3 ].
            _Error: 'Unknown state'.
        ).

        total
    |
        total: 0.

        [| :break |
            input each: [| :line. him. you |
                line = '' ifTrue: break.

                line: line splitBy: ' '.
                him: line at: 0.
                you: line at: 1.

                total: total + (bonusFor: winnerOf: him VS: you) + (scoreFor: you).
            ].
        ] break.

        total
    ).

    part2 = (|
        "Rock = 1, Paper = 2, Scissors = 3"
        expectedScoreFor: him VS: you = (
            you = 'Z' ifTrue: [
                him = 'A' ifTrue: [ ^ 2 ].
                him = 'B' ifTrue: [ ^ 3 ].
                him = 'C' ifTrue: [ ^ 1 ].
            ].
            you = 'Y' ifTrue: [
                him = 'A' ifTrue: [ ^ 1 ].
                him = 'B' ifTrue: [ ^ 2 ].
                him = 'C' ifTrue: [ ^ 3 ].
            ].
            you = 'X' ifTrue: [
                him = 'A' ifTrue: [ ^ 3 ].
                him = 'B' ifTrue: [ ^ 1 ].
                him = 'C' ifTrue: [ ^ 2 ].
            ].
            _Error: 'Unknown state'.
        ).

        scoreFor: you = (
            you = 'Z' ifTrue: [ ^ 6 ].
            you = 'Y' ifTrue: [ ^ 3 ].
            you = 'X' ifTrue: [ ^ 0 ].
            _Error: 'Unknown state'.
        ).

        total.
    |
        total: 0.

        [| :break |
            input each: [| :line. him. you |
                line = '' ifTrue: break.

                line: line splitBy: ' '.
                him: line at: 0.
                you: line at: 1.

                total: total + (expectedScoreFor: him VS: you) + (scoreFor: you).
            ].
        ] break.

        total
    ).

    run = (
        input: (
            std file open: 'input.txt'
                ; readAll
                ; splitBy: '\n'
        ).

        part1 asString printLine.
        part2 asString printLine.
    )

|) run.
```

</details>
<br />

As you can see, today's theme was focused more on the power of inline methods.
They allow you, in essence, to define utility methods which you can use as part
of the method itself. They're treated specially in the VM such that when you
send that message, it is sent on the current activation object, and can
therefore access things like method slots. They're really useful for defining
one-off functions, the recursive case for recursive algorithms, etc.

Yesterday [I got rid
of](https://github.com/sin-ack/zigself/commit/0ee45b57aad62c0ea1d2e0f67925d7eecd560da1)
a useless operation, which shaved off 20% runtime. On my laptop (slightly slower
than the machine I ran yesterday's code on), the two parts complete in 1.6
seconds. Of course, there's still a ton of room for improvement both on the Self
side and the VM side.

No AoC-specific ZigSelf commits this time, since today's problem can be solved
as-is. :^)

---

<br />

Join the [ZigSelf Discord server](https://discord.gg/HJ62kw6yvn) in order to
discuss the language and/or ask me questions!
