---
title: "A look at Self's object system"
date: 2021-03-29T13:00:00+02:00
draft: false
tags:
  - self
  - vm
categories:
  - self

type: post
---

When I [first introduced Self](/posts/a-tour-of-self/), I mentioned that Self is
a programming language which employs _prototype-based inheritance_; it does not
depend on a class-instance distinction to facilitate object-oriented
programming, but rather, it uses a _prototype object_ from which copies are
created. The methods associated with a prototype is stored in a _traits object_
which is just another object storing methods that can be used on a prototype and
all its copies through a parent slot.

The trouble begins when we actually try to implement this prototype-based
inheritance system as an actual virtual machine, particularly with regards to
memory consumption. Say that I have a prototype object `point` with a constant
parent slot pointing to `traits point` and two assignable slots, `x` and
`y`.[^1] When we copy this object, we would expect to copy all three fields. But
if we scale that up, that's so much unnecessary data being copied! For one, for
every copy of the `point` prototype we make, we're copying that `traits point`
parent slot reference. Not to mention, because our language is dynamic, we can't
simply treat `x` and `y` as offsets into memory, because they might be constant
or assignable (Self can't intrinsically know). So our naive implementation would
be pretty inefficient.

Can we make this more memory-efficient? In fact, we can; we can exploit the fact
that most Self objects that are copied only ever have their assignable slots
modified. Let's take a look at how the original developers of Self have
optimized the memory usage of the Self virtual machine.

<!--more-->

A little heads-up before we begin: This one's going to be a less hands-on post
as I'm mainly going to be talking about implementation details of the VM itself
rather than the Self programming environment.

## The key insight

Just in the previous paragraph I mentioned that most objects in Self only ever
have their assignable slots modified. In fact, this isn't just some rule of
thumb: there is no assignment slot for constant slots, so you cannot modify the
constant slots of an object through regular methods like you would with
assignable slots[^2].

So with that in mind, we could just reduce the copy of an object to the contents
of the assignable slots and keep the rest of the object itself as a separate
"information" structure that tells the VM what is supposed to be what.

And that's exactly what the original developers of the Self VM did. They called
their invention an "object map". An _object map_ (called just "map" hereafter)
stores all the information the Self VM needs to know about an object, such as
the parent slots, constant slots, etc. Using the map, we can reduce the amount
of memory we require for a copy of our `point` prototype object to just two
memory words (`x` and `y`) plus some book-keeping to point to the map itself.

## Self object references

For things like integers and floating point numbers, even the book-keeping would
be too much memory. For a 64-bit integer, just adding a pointer to a map doubles
our memory footprint! So, the authors of Self turned to good-old pointer tagging
for object references.

In the Self virtual machine, all object references have their bottom two bits
used as a tag. The possible values are as follows.

- Integers are given `00`, because that makes adding, subtracting and comparing
  them a single assembly instruction (because there's no bitwise ops
  required)[^3].
- For object references, we can exploit the fact that all our objects will be
  aligned to the nearest machine word. They are tagged by simply binary ORing
  the bottom two bits with `01`. Then they can be ANDed out when the pointer to
  the memory space needs to be dereferenced.
- Floats are marked with `10`. Since the bottom two bits of the float is part of
  the mantissa, we can simply AND the bits away at the cost of a little
  precision loss.
- Finally, _object markers_ are tagged with `11`. We will talk about object
  markers shortly.

Of course, this means that our integers are 62-bit and our floats lose some
precision in their last decimal places, but it is probably better than carrying
around an additional 8 bytes (more if we want to mark our objects).

Note that newer techniques like NaN boxing are not used in Self. There are a
couple reasons for this: first, from what I can tell, NaN boxing became more
widely used in the early-to-mid 90s while Self's design was created in the late
80s; and second, Self's dynamic inheritance and dynamic typing means that most
objects we would optimize by boxing into NaNs require carrying around a parent
slot pointer (48 bits in modern systems and possible 64 bits in the future),
leaving not enough space for other attributes (i.e. the size value in vectors).

## Self objects in memory

So what do Self objects in memory actually carry? Well, they consist of the
following:

- A machine word for the _object marker_ that we talked about above. Object
  markers are used to mark the start of an object.
- A machine word for the _map reference_ for this object. It's simply a tagged
  pointer pointing to a map.
- If the object is a vector or byte vector object, an additional machine word is
  added for the number of items.
- If the object is a slots object (i.e. contains just slots), each assignable
  slot follows the first two words. If the object is a vector, `n` machine words
  for the items follow instead. If the object is a byte vector, a pointer to the
  _bytevector space_ (explained in the next section) is stored.

That's all we need in order to represent a Self object with our current scheme.
The first word contains a bit more information identifying the _type_ of the
object (slots, vector, bytevector, map) as well as some implementation specific
GC information.

## Self maps in memory

Now that we described the objects themselves, let's look at maps, which are the
descriptor structures for the objects in memory. They contain the following:

- An object marker, just like regular objects. However, they are identified as
  _maps_ instead of any object type.
- A map reference. In Self, all maps are required to have a map reference
  pointing to the _map-map_, which is the map that references itself. The V8
  Javascript engine calls this the _meta-map_ which I like more.
- In the third word, there is the "garbage collection link" which creates a
  linked list out of all the maps for scavenging. There's also a vtable pointer
  in the fourth word but that's an implementation detail and can be ignored.
- The total size of the object in machine words. Since Self was designed as a
  32-bit system, that's just the total size of the map divided by four.
- A few pointers to _dependency links_. Dependency links are used a lot in the
  compiler which is out of scope for this article. They're used to track changes
  for invalidation of compiled code when an object changes.
- If this map is for a method slot, then there is a machine word for the object
  reference to the bytecode.
- Following that are "slot descriptor" objects. They contain the name, type and
  either the contents (for constant slots) or the byte offset from the object
  (for mutable and assignment slots) of the slot. Vector maps only have a parent
  slot[^5].

## The Self memory space

The Self object system also splits Self objects and byte-vector values into two
separate sides of the address space. [The implementation
paper](https://raw.githubusercontent.com/russellallen/self/master/docs/papers/implementation.pdf)
describes that this was done to improve the speed of scavenging[^4]. Since byte
vectors (i.e. strings) can have any bit pattern, it would be quite easy to
masquerade a bytevector's contents as an object marker by simply setting the
bottom two bits of a word to `11`. Smalltalk interpreters at the time solved
this by iterating over the memory space object-by-object (i.e. doing bounds
checking for every object) but that causes performance problems. Self was able
to achieve double the speed (3MB/s at the time, according to the paper) by
_segregating_ the raw byte data to the upper area of the allocated address
space. This allows the scavenger to simply scan the entire object side of the
memory for the `11` marker instead of caring about the length of the object, for
example.

## A more modern implementation: V8

The above information was about how Self implements its objects in the memory
space. This particular implementation was conceived in the late 80s. However, as
we all know, there is a newer and more popular language that also uses
prototype-based object-orient programming: Javascript. I wanted to know what
sort of object implementation Javascript had in a modern engine, so I started to
research how the V8 Javascript engine does it. While researching, I came across
this [helpful article by Jay
Conrod](https://www.jayconrod.com/posts/52/a-tour-of-v8-object-representation)
which you should check out.

Something that surprised me was how similar V8's object representation was to
Self's object representation. Perhaps the authors were influenced by the paper,
or perhaps they simply came up with a similar representation on their own.
However, V8 seems to have a few key differences with regards to the object
representation.

### Map transitions

In its current implementation, Self will create a new map everytime an object
has slots added or one of its constant slots modified. This is understandable,
since that makes an object diverge from others. In Self this is mostly okay,
because objects are copied wholesale from prototypes and all the properties that
an object has is known at copy-time.

However, Javascript has a different model, particularly for constructor
functions. When a new object is created, it is usually assigned values
one-by-one. Using Self's implementation, this would mean that a new map would be
created everytime. Not only that, but many duplicate maps would be created from
the re-assignments. To combat this, the V8 engine employs _map transitions_.

I'll take the example from Jay Conrod's article. When we have a constructor like
this:

```javascript
function Point(x, y) {
  this.x = x;
  this.y = y;
}
```

The V8 engine can create the following maps when this code is first executed:

```
Map0: # Empty map
  - x: Transition(Map1)

Map1:
  - x: Field
  - y: Transition(Map2)

Map2:
  - x: Field
  - y: Field
```

When similar code is executed, this will slowly create a tree from the root map
to the leaves which are concrete object maps.

Now, to be clear, this method is neither superior or inferior when compared to
Self's method of just allocating a new map. In particular, in the example Map1
cannot be de-allocated because new Point objects will be using that map when
being constructed. Self does not need something like this, because most objects
are constructed at once with all their properties and copied many times.

### Constant Method Fields

Another Javascript-related optimization that V8 does is to inline function
pointers as constant. This seems to be very similar to how Self holds the values
of constant slots directly in the slot descriptors.

Here I'm taking another example from the V8 article. Suppose we had a
constructor like this:

```javascript
function Point(x, y) {
  this.x = x;
  this.y = y;
  this.distance = PointDistance;
}

function PointDistance(p) {
  var dx = this.x - p.x;
  var dy = this.y - p.y;
  return Math.sqrt(dx * dx + dy * dy);
}
```

Because we are assigning the same constant function everytime, it would not make
sense to store a reference to it in every Point instance. So the V8 runtime will
store the pointer as a constant directly in the map instead, like so:

```
...

Map2:
  - x: Field
  - y: Field
  - distance: Transition(Map3)

Map3:
  - x: Field
  - y: Field
  - distance: ConstantFunction(PointDistance)
```

The optimizer would then replace any references to this constant function with a
direct reference to the function itself, eliminating indirection.

This is required in Javascript, because objects are malleable during runtime,
and new properties may be added to them at anytime; in addition, in Javascript
object properties cannot be constant unless the object is frozen. In contrast,
Self already has this feature, however it is implemented in a different way:
Because objects in normal operation are either created from scratch (with all
their properties) or cloned, the constant values are immediately embedded into
the newly created map without the need for transitions or special references to
constant functions. The bytecode compiler can then inline these constant values
just as it would with any other constant value. In Self, the Point prototype
would look like this:

```ruby
(|
  parent* = traits clonable.

  x <- 0.
  y <- 0.

  copyX: x Y: y = (| c |
    c: self clone.
    c x: x.
    c y: y.
    c
  ).

  distance: p = (| dx. dy |
    dx: x - p x.
    dy: y - p y.
    (dx square + dy square) squareRoot
  ).
|)
```

Since all the constant values (`parent`, `copyX:Y:` and `distance:`) are
already known, the Self VM can create a single Map object which will embed a
reference to the methods directly.

Note that similar to the Javascript prototype object, in Self a _traits object_
is used to store methods that can be used on a prototype and its copies. While
this is purely an idiom, it is a good idiom nevertheless. In particular, when
new methods are added to the traits object, the existing copies can also benefit
from it.

## Conclusion

Self's object system still seems to hold up since its initial implementation
paper from 1989. Because the properties of Self allow it to allocate a single
map for objects and then keep it around for a long time, a feature like map
transitions does not seem to be necessary. Similarly, constant references in
maps have existed from the beginning, because Self expects the programmer to
define all slots during object creation, and makes a distinction between
constant and assignable slots.

There seems to be a key difference in how Javascript and Self treat the
prototype-based inheritance system too, from what I can tell: In Javascript the
`prototype` seems to have become a place to put the methods of an object. In a
way, it's just the "class" object named a different way. In Self, the prototype
object is the object that other code clones in order to get an "instance". You
could say that the prototype is the "default value" of an object.

If we were to write the Javascript `Point` code above in a Self-like style, it
would look like this:

<details>
<summary>Self-like Javascript</summary>

```javascript
var Point = {
  x: 0,
  y: 0,

  clone(x, y) {
    var p = Object.create(this);
    p.x = x;
    p.y = y;
    return p;
  },

  distance(p) {
    var dx = this.x - p.x;
    var dy = this.y - p.y;
    return Math.sqrt(dx * dx + dy * dy);
  },

  toString() {
    return "Point(" + this.x + ", " + this.y + ")";
  },
};

// To use it:
var p = Point.clone(3, 4);
console.log(p); // Point(3, 4)
console.log(p.distance(Point)); // 5
```

Pretty similar, eh?

</details><br />

Thanks for reading. This article wasn't as exciting as the previous ones,
however I think it's still pretty interesting to look at how Self handles
objects in memory and how it compares to newer programming languages in a
similar paradigm. Also, sorry for being absent for almost 2 months from here. I
had some IRL business to take care of, however that is done and now I should be
able to post much more often. Stay tuned!

<!-- prettier-ignore -->
[^1]: This is a simplified version of the initial object problem posed by the
  paper.

<!-- prettier-ignore -->
[^2]: Keyword regular. Using primitives and mirrors, you can of course change an
  object's constant slots but that has additional side effects, like creating a
  completely new map.

<!-- prettier-ignore -->
[^3]: Multiplication requires one right shift, and division requires two right
  shifts before and a left shift after.

<!-- prettier-ignore -->
[^4]: We will look at Self's GC system in a future post. For now, just know that
  scavenging is a quick GC technique that is faster than a full GC cycle.

<!-- prettier-ignore -->
[^5]: Curiously, the current Self implementation does allow more than just the
  parent slot on vector objects, so maybe this restriction was lifted after
  the implementation paper was published.
