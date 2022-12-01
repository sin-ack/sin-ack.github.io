---
title: "AoC 2022 in ZigSelf, day 1"
date: 2022-12-01T06:00:00Z
tags:
  - zigself
  - advent-of-code
categories:
  - zigself
type: post
---

Hi there. I had mentioned that I would be solving AoC puzzles this year with
ZigSelf. I plan to commit to it as much as possible, but no promises. :^)

The first day of AoC has always been quite easy, and today is no exception, so
this was rather simple.

<!--more-->

You can see the [source code
here](https://github.com/sin-ack/zigself-advent-of-code/blob/893eb9f51c2d925ba41076382187812bb7cd2f61/2022/day1/solve.self),
or in the below section.

<details>
<summary>Source code listing</summary>

```smalltalk
'../../../objects/everything.self' _RunScript.

(|
    parent* = std traits singleton.

    "Put input.txt in your cwd"
    input.

    eachSum: block = (
        input splitBy: '\n\n'; each: [| :group. sum |
            sum: 0.
            [| :break |
                group splitBy: '\n'; each: [| :item |
                    item = '' ifTrue: break.
                    sum: sum + item asInteger.
                ].
            ] break.

            block value: sum.
        ].
    ).

    part1 = (| maxCalories. |
        maxCalories: 0.

        eachSum: [| :sum |
            sum > maxCalories ifTrue: [ maxCalories: sum ].
        ].

        maxCalories.
    ).

    part2 = (| calorieTotals. |
        calorieTotals: std vector copyRemoveAll.

        eachSum: [| :sum |
            calorieTotals add: sum.
        ].

        calorieTotals sort.
        calorieTotals reverse.
        calorieTotals copyFrom: 0 To: 3; fold: [| :acc. :item | acc + item] Initial: 0.
    ).

    run = (
        input: (std file open: 'input.txt'; readAll).
        part1 asString printLine.
        part2 asString printLine.
    ).
|) run.
```

</details>
<br />

The main problem I had was that we were missing various vector primitives such
as sorting or reversing. Those are now in master as part of `std vector`. They
will eventually be lifted up to the collection level so that every collection
benefits from them; however, the main issue there is the fact that, for example,
the sorting operation requires a collection to both be mutable and indexable,
and we don't have a mixin type that adds both of them together yet.

Additionally, performance is not very good overall, particularly because no
attention to optimization has been given yet. On release mode, calculating the
result with my input takes 1.30s on my machine.

ZigSelf commits:

- [objects: Implement std vector sort](https://github.com/sin-ack/zigself/commit/771b1dc1d5b2a1acea48a89c06d35328480eb222)
- [objects: Implement std vector reverse](https://github.com/sin-ack/zigself/commit/bcb088de8db749b112bedf3a3f51d572cb39b7eb)
- [objects: Implement std vector copyFrom:To:](https://github.com/sin-ack/zigself/commit/632a5b584ae467c36bcab9ecc200476d1ff2b132)
