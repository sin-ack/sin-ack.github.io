---
title: "How malloc broke Serenity's JPGLoader, or: how to win the lottery"
date: 2021-06-03T00:00:00+02:00
tags:
  - serenity
  - bug
categories:
  - serenity

type: post
---

I got the chance to investigate an interesting bug in
[SerenityOS](https://serenityos.org/) this week. It was related to the decoding
of JPG images in the operating system. For some reason, when a JPG image is
viewed, it comes out like this:

{{< figure src="/images/serenity-jpgloader/lenna-broken.png" caption=`Lenna,
showing up with incorrect colors.` >}}

Weird, huh? Also seems like a simple confusion of RGB vs. BGR. And sure enough,
making the following change on `JPGLoader.cpp`:

```diff
-   const Color color { (u8)block.y[pixel_index], (u8)block.cb[pixel_index], (u8)block.cr[pixel_index] };
+   const Color color { (u8)block.cr[pixel_index], (u8)block.cb[pixel_index], (u8)block.y[pixel_index] };
    context.bitmap->set_pixel(x, y, color);
```

makes the image show up correctly. Case closed!

...not. Why did this even break in the first place?<!--more--> The last non-reverted change
to `JPGLoader.cpp` is reported by Git to be over a month ago:

{{< figure src="/images/serenity-jpgloader/commitlog.png" caption=`Commit log
at the time of JPGLoader being broken.` >}}

And I remembered very well that JPG images worked just fine about a week or two
ago, as I had set a JPG image as my background and would've noticed if it looked
wrong.

Well, time to bisect! I didn't know when to start, so I picked the last 1000
commits (where images showed up correctly), and started bisecting.

## Bisect hell

Please skip to the next section if you'd like to avoid C++ whining.

SerenityOS, being an operating system project that focuses on doing its own
thing, also has its own standard library called AK (which stands for
<span title="Andreas Kling" style="border-bottom: 1px dotted">Agnostic Kit</span>).
This library is analogous to
C++'s STL, but is more readable due to not having to support a myriad of
different operating systems and not having to contort oneself to conform to
[hideous coding standards](https://www.gnu.org/prep/standards/).

One of the nice things about having the standard library in the same repository
as its users is that making changes is very easy as the change propagates to
everyone who pulls from master. However, this is a double edged sword when it
comes to C++; because _everyone_ includes the standard library (even if you
don't include it, your includes will), and because C++'s template system means
that everything that's templated has to include the definitions in the header as
well, this means that _anytime_ someone touches AK in a commit, the _entire_
operating system has to be rebuilt (~3400 files at the time of writing).
`ccache`, while being useful in many situations, cannot handle this case.
Additionally, due to the breakneck pace of the SerenityOS project, someone ends
up touching AK at least once every 100 commits or so.

As a result, during the 1000 commits I ended up bisecting for, I had to build
SerenityOS from scratch about 4-5 times on a 2011 laptop with Sandy Bridge
Mobile. While this isn't the fault of the project, I'm still mad.

## Bisect results

So, after bisecting 1000 commits, rebuilding the OS from scratch several times
and pulling my hair out because I didn't understand how bisect worked, I
_finally_ found the commit that broke JPG images. Drumroll please...

```
f89e8fb71a4893911ee5125f34bd5bbb99327d33
Author:     Gunnar Beutner
AuthorDate: Sat May 15 10:06:41 2021 +0200

AK+LibC: Implement malloc_good_size() and use it for Vector/HashTable

This implements the macOS API malloc_good_size() which returns the
true allocation size for a given requested allocation size. This
allows us to make use of all the available memory in a malloc chunk.

For example, for a malloc request of 35 bytes our malloc would
internally use a chunk of size 64, however the remaining 29 bytes
would be unused.

Knowing the true allocation size allows us to request more usable
memory that would otherwise be wasted and make that available for
Vector, HashTable and potentially other callers in the future.
```

Uh, sorry, what?

But it was. Building the commit right before this one showed the image
correctly:

{{< figure src="/images/serenity-jpgloader/lenna-beforebroken.png"
caption=`Lenna, before it was broken.` >}}

Initial discussion with other developers made me think that either `JPGLoader`
or something else up the chain is depending on the capacity of a `Vector` and
writing directly into it when it really shouldn't. So I began hunting down
possible causes.

## A surprising discovery

The commit seemed to touch the two main container types: `HashTable` (which
`HashMap` depends on) and `Vector`. Both are used in the `JPGLoader` code, and
either could be the cause of the problem here.

I picked `HashTable` at random, removed the offending line:

```diff
         new_capacity = max(new_capacity, static_cast<size_t>(4));
-        new_capacity = kmalloc_good_size(new_capacity * sizeof(Bucket)) / sizeof(Bucket);

         auto* old_buckets = m_buckets;
```

and rebuilt the system, while joking around in chat about how this can't
possibly be the problem.

...but then it fixed the issue.

What? How? Why does the `HashTable` capacity being different matter?! `HashTable`
isn't even a contiguous stream of data you can write to, so you shouldn't even
be able to assume its capacity!

Before I present the full story to you, I'll have give a brief background on how
`JPGLoader` used to work.

## Non-deterministic serial component iteration

That's really the most appropriate title I can give this section.

`JPGLoader` previously would read information about a JPG component from the
"Start of Frame" section of the JPG file into a struct called `Component`, and
then store that in a `HashTable`. Of course, the order in a JPG file for each
component should always be `Y`, `Cb` and `Cr`, so the `Component` struct would
idiosyncratically carry a `serial_id`, which was the position of the `Component`
within the file. The reason the `Component`s were in a hash table was that they
would then be checked against the component ordering in a "Start of Scan"
section to make sure all the components in the SOS section are in the expected
order. Why this code was written this was instead of just checking against the
ID by linearly iterating over the `Component`s, I have no idea.

Anyway, these components would then be iterated over during the different
decoding stages of `JPGLoader`, during which the component information would be
used to perform transforms on macroblocks.

## Getting close

When I added some debug prints to see how the components were read, I saw this
in the commit with the broken colors:

```
ImageDecoder(33:33): Looking at component 0
ImageDecoder(33:33): Looking at component 2
ImageDecoder(33:33): Looking at component 1
ImageDecoder(33:33): Looking at component 0
ImageDecoder(33:33): Looking at component 2
ImageDecoder(33:33): Looking at component 1
...
```

And when I checked out the previous commit, I saw this:

```
ImageDecoder(33:33): Looking at component 0
ImageDecoder(33:33): Looking at component 1
ImageDecoder(33:33): Looking at component 2
ImageDecoder(33:33): Looking at component 0
ImageDecoder(33:33): Looking at component 1
ImageDecoder(33:33): Looking at component 2
...
```

The final piece of the puzzle: During the discussion of this bug with
[CxByte](https://twitter.com/the_semicolon_) at my wit's end, we ended up
manually messing with the order of the components to see what would happen, and
got this message:

```
ImageDecoder(32:32): Huffman stream exhausted. This could be an error!
ImageDecoder(32:32): Failed to build Macroblock 3277
```

...ah. Of course. It's a stream.

## The bug

So, here's a quick rundown of the bug:

- Someone used a `HashTable` to store objects that should be ordered, then
  iterated over it using the basic `HashTable` iterator
- The hash of the component IDs in the JPG files were passed into `int_hash`
  for hash table bucket selection
- Not only did they get _just the right value_ to be in order, they got
  inserted into a HashTable with _just the right amount_ of buckets to be in
  the correct order
- This caused the Huffman stream to be read in the correct order for each
  component, thereby masking the bug
- This bug was masked since `JPGLoader`'s inception by sheer luck until someone
  messed with the size of the `HashTable`

## The fix

And finally, at the end of about 10 hours of debugging, [here is the
commit](https://github.com/SerenityOS/Serenity/commit/a10ad24c760bfe713f1493e49dff7da16d14bf39)
that fixed this monster of a bug:

```
a10ad24c760bfe713f1493e49dff7da16d14bf39
Author:     sin-ack
AuthorDate: Mon May 31 15:22:04 2021 +0000
Commit:     Linus Groh
CommitDate: Mon May 31 17:26:11 2021 +0100

LibGfx: Make JPGLoader iterate components deterministically

JPGLoader used to store component information in a HashTable, indexed
by the ID assigned by the JPEG file.  This was fine for most purposes,
however after f89e8fb7 this was revealed to be a flawed implementation
which causes non-deterministic iteration over components.

This issue was previously masked by a perfect storm of int_hash being
stable for the integer values 0, 1 and 2; and AK::HashTable having just
the right amount of buckets for the components to be ordered correctly
after being hashed with int_hash. However, after f89e8fb7,
malloc_good_size was used for determining the amount of space for
allocation; this caused the ordering of the components to change, and
images started showing up with the red and blue channels reversed. The
issue was finally determined to be inconsistent ordering after randomly
changing the order of the components caused Huffman decoding to fail.

This was the result of about 10 hours of hair-pulling and repeatedly
doing full rebuilds due to bisecting between commits that touched AK.
Gunnar, I like you, but please don't make me go through this again. :^)

Credits to Andrew Kaster, bgianf, CxByte and Gunnar for the debugging
help.
```

## Final thoughts

Sometimes the simplest problems might point at big mistakes within. I could've
probably fixed this by just swapping the order of the arguments right then and
there, and it would've worked; until someone else came along and changed the
order again. Thankfully, now we will be able to look at tubas with correct
colors in peace.

{{< figure src="/images/serenity-jpgloader/tuba.png" caption=`A tuba with the
correct colors. Source: music123.com` >}}

## Thanks

Thanks to CxByte, Gunnar, Andrew and Brian for their help with debugging this,
and their helpful tips. Gunnar in particular was the one who uncovered this bug,
and despite my satirical jab in the commit message helped uncover this very
interesting bug, so he's the one who made this post possible.

Also, thanks to the person who introduced this bug (the commit log gets a little
fuzzy, so I'm not quite sure who did) and hope he buys a lottery ticket. :^)

And thank you for reading. I'll probably post sometime in the future, but work's
been keeping me busy. But maybe I'll find another bug to suck me into a rabbit
hole. Stay tuned!
