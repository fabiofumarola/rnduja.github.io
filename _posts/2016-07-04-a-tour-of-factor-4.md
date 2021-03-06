---
layout:     post
title:      "A tour of Factor: 4"
subtitle:   "Object orientation"
date:       2016-07-04 12:00:00
author:     "Andrea Ferretti"
header-img: "img/chain.jpg"
comments:    true
tags:       [concatenative-programming]
---

The object system and protocols
-------------------------------

Although it is not apparent from what we have said so far, Factor has
object-oriented features, and many core words are actually method invocations.
To better understand how objects behave in Factor, a quote is in order:

> I invented the term Object-Oriented and I can tell you I did not have C++ in mind.
> Alan Kay

The term object-oriented has as many different meanings as people using it. One
point of view - which was actually central to the work of Alan Kay - is that it
is about late binding of function names. In Smalltalk, the language where this
concept was born, people do not talk about calling a method, but rather sending
a message to an object. It is up to the object to decide how to respond to this
message, and the caller should not know about the implementation. For instance,
one can send the message `map` both to an array and a linked list, but
internally the iteration will be handled differently.

The binding of the message name to the method implementation is dynamic, and
this is regarded as the core strenght of objects. As a result, fairly complex
systems can evolve from the cooperation of independent objects who do not mess
with each other internals.

To be fair, Factor is very different from Smalltalk, but still there is the
concept of classes, and generic words can defined having different
implementations on different classes.

Some classes are builtin in Factor, such as `string`, `boolean`, `fixnum` or
`word`. Next, the most common way to define a class is as a **tuple**. Tuples
are defined with the `TUPLE:` parsing word, followed by the tuple name and the
fields of the class that we want to define, which are called **slots** in
Factor parlance.

Let us define a class for movies:

```factor
TUPLE: movie title director actors ;
```

This also generates setters `>>title`, `>>director` and `>>actors` and getters
`title>>`, `director>>` and `actors>>`. For instance, we can create a new movie
with

```factor
movie new "The prestige" >>title
  "Christopher Nolan" >>director
  { "Hugh Jackman" "Christian Bale" "Scarlett Johansson" } >>actors
```

We can also shorten this to

```factor
"The prestige" "Christopher Nolan"
{ "Hugh Jackman" "Christian Bale" "Scarlett Johansson" }
movie boa
```

The word `boa` stands for 'by-order-of-arguments' and is a constructor that
fills the slots of the tuple with the items on the stack in order. `movie boa`
is called a **boa constructor**, a pun on the Boa Constrictor. It is customary
to define a most common constructor called `<movie>`, which in our case could
be simply

```factor
: <movie> ( title director actors -- movie ) movie boa ;
```

In fact, boa constructor are so common, that the above line can be shortened to

```factor
C: <movie> movie
```

In other cases, you may want to use some defaults, or compute some fields.

The functional minded will be worried about the mutability of tuples. Actually,
slots can be declared to be read-only with `{ slot-name read-only }`. In this
case, the field setter will not be generated, and the value must be set at the
beginning with a boa constructor. Other valid slot modifiers are `initial:` -
to declare a default value - and a class word, such as `integer`, to restrict
the values that can be inserted.

As an example, we define another tuple class for rock bands

```factor
TUPLE: band
  { keyboards string read-only }
  { guitar string read-only }
  { bass string read-only }
  { drums string read-only } ;
: <band> ( keyboards guitar bass drums -- band ) band boa ;
```

together with one instance

```factor
"Richard Wright" "David Gilmour" "Roger Waters" "Nick Mason" <band>
```

Now, of course everyone knows that the star in a movie is the first actor,
while in a rock band it is the bass player. To encode this, we first define a
**generic word**

```factor
GENERIC: star ( item -- star )
```

As you can see, it is declared with the parsing word `GENERIC:` and declares
its stack effects but it has no implementation right now, hence no need for the
closing `;`. Generic words are used to perform dynamic dispatch. We can define
implementations for various classes using the word `M:`

```factor
M: movie star actors>> first ;
M: band star bass>> ;
```

If you write `star .` two times, you can see the different effect of calling a
generic word on instances of different classes.

Builtin and tuple classes are not all that there is to the object system: more
classes can be defined with set operations like `UNION:` and `INTERSECTION:`.
Another way to define a class is as a **mixin**.

Mixins are defined with the `MIXIN:` word, and existing classes can be added to
the mixin writing

```factor
INSTANCE: class mixin
```

Methods defined on the mixin will then be available on all classes that belong
to the mixin. If you are familiar with Haskell typeclasses, you will recognize
a resemblance, although Haskell enforces at compile time that instance of
typeclasses implement certain functions, while in Factor this is informally
specified in documentation.

Two important examples of mixins are `sequence` and `assoc`. The former defines
a protocol that is available to all concrete sequences, such as strings, linked
lists or arrays, while the latter defines a protocol for associative arrays,
such as hashtables or association lists.

This enables all sequences in Factor to be acted upon with a common set of
words, while differing in implementation and minimizing code repetition
(because only few primitives are needed, and other operations are defined for
the `sequence` class). The most common operations you will use on sequences
are `map`, `filter` and `reduce`, but there are many more - as you can see with
`"sequences" help`.

Learning the tools
------------------

A big part of the productivity of Factor comes from the deep integration of the
language and libraries with the tools around them, which are embodied in the
listener. Many functions of the listener can be used programmatically, and vice
versa. You have seen some examples of this:

* the help is navigable online, but you can also invoke it with `help` and print
  help items with `print-content`;
* the `F2` shortcut or the words `refresh` and `refresh-all` can be used to
  refresh vocabularies from disk while continuing working in the listener;
* the `edit` word gives you editor integration, but you can also click on file
  names in the help pages for vocabularies to open them.

The refresh is actually quite smart. Whenever a word is redefined, words that
depend on it are recompiled against the new defition. You can check by yourself
doing

```factor
: inc ( x -- y ) 1 + ;
: inc-print ( x -- ) inc . ;
5 inc-print
```

and then

```factor
: inc ( x -- y ) 2 + ;
5 inc-print
```

This allows you to always keep a listener open, improving your definitions,
periodically saving your definitions to file and refreshing, without ever
having to reload Factor.

You can also save the whole state of Factor with the word `save-image` and later
restore it by starting Factor with

```factor
./factor -i=path-to-image
```

In fact, Factor is image-based and only uses files when loading and refreshing
vocabularies.

The power of the listener does not end here. Elements of the stack can be
inspected by clicking on them, or by calling the word `inspector`. For instance
try writing

```factor
TUPLE: trilogy first second third ;
: <trilogy> ( first second third -- trilogy ) trilogy boa ;
"A new hope" "The Empire strikes back" "Return of the Jedi" <trilogy>
"George Lucas" 2array
```

You will get an item that looks like

```factor
{ ~trilogy~ "George Lucas" }
```

on the stack. Try clicking on it: you will be able to see the slots of the
array and focus on the trilogy or on the string by double-clicking on them.
This is extremely useful for interactive prototyping. Special objects can
customize the inspector by implementing the `content-gadget` method.

There is another inspector for errors. Whenever an error arises, it can be
inspected with `F3`. This allows you to investigate exceptions, bad stack
effects declarations and so on. The debugger allows you to step into code,
both forwards and backwards, and you should take a moment to get some
familiarity with it. You can also trigger the debugger manually, by entering
some code in the listener and pressing `Ctrl+w`.

Another feature of the listener allows you to benchmark code. As an example, we
write an intentionally inefficient Fibonacci:

```factor
DEFER: fib-rec
: fib ( n -- f(n) ) dup 2 < [ ] [ fib-rec ] if ;
: fib-rec ( n -- f(n) ) [ 1 - fib ] [ 2 - fib ] bi + ;
```

(notice the use of `DEFER:` to define two mutually recursive words). You can
benchmark the running time writing `40 fib` and then pressing Ctrl+t instead of
Enter. You will get timing information, as well as other statistics.
Programmatically, you can use the `time` word on a quotation to do the same.

You can also add watches on words, to print inputs and outputs on entry and
exit. Try writing

```factor
\ fib watch
```

and then run `10 fib` to see what happens. You can then remove the watch with
`\ fib reset`.

Another very useful tool is the `lint` vocabulary. This scans word definitions
to find duplicated code that can be factored out. As an example, let us define
a word to check if a string starts with another one. Create a test vocabulary

```factor
"lintme" scaffold-work
```

and add the following definition

```factor
USING: kernel sequences ;
IN: lintme

: startswith? ( str sub -- ? ) dup length swapd head = ;
```

Load the lint tool with `USE: lint` and write `"lintme" lint-vocab`. You will
get a report mentioning that the word sequence `length swapd` is already used
in the word `(split)` of `splitting.private`, hence it could be factored out.

Now, you would not certainly want to modify the source of a word in the
standard library - let alone a private one - but in more complex cases the
lint tool is able to find actual repetitions. It is a good idea to lint your
vocabularies from time to time, to avoid code duplication and as a good way
to discover library words that you may have accidentally redefined.

Finally, there are a few utilities to inspect words. You can see the
definition of a word in the help tool, but a quicker way can be `see`.
Or, vice versa, you may use `usage.` to inspect the callers of a given
word. Try `\ reverse see` and `\ reverse usage.`.

In the next post, we will start doing some metaprogramming in Factor.

Until then!