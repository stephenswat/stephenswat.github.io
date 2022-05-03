---
title: Writing a Parser Combinator Library from Scratch in Haskell
date: 2021-12-25
author: Stephen Nicholas Swatman
published: true
---

There are few things more satisfying and educational for novice Haskell
developers (like me!) than writing your own parser combinator. It's a fantastic
exercise in reasoning about functions, functors, monads, and many other
essential parts of the Haskell programming language. Best of all: it's not that
difficult! In this walkthrough, we'll be building our own parser combinator
library from the ground up, and I will do my best do explain it in a way that
is accessible to people with very little knowledge about the concepts I
mentioned earlier.

## Parser combinators

A parser is a computer program that takes a string (usually a string of
characters, but also strings of tokens or other data types) and converts it to
some computationally meaningful structure. Parsers are monumentally important
in computing, as they allow us to read in structured data. In fact, parsers
make programming in most high-level programming languages possible by allowing
compilers and interpreters to translate between human-readable syntax and data
structures that can be used for compilation.

Parsers come in many flavours. When I first encountered parsers of the generic
kind, I was taking a compiler construction course at university and we built a
parser using a _parser generator_ program called _bison_. A parser generator
takes a _grammar_ which describes the rules of a language and outputs some
(often human-unreadable) code that is able to parse it. A parser _combinator_
is written in a programming language of your choice (as opposed to a
domain-specific language for describing grammars), and works by _combining_
smaller parsers. For example, we might want to build a parser for a language
consisting of a series of tuples:

```
(4,5)
(1,10)
(5,-2)
...
(7, 8)
```

To build a parser combinator for this language, we can combine parsers for
individual characters and numbers. For example, we might start out with a
simple function to parse one character. From there, we can construct a parser
to parse a digit. From there, we might combine a parser for a negative sign
(`-`) with our parser for digits to create a parser for numbers. We can combine
that parser, in turn, with parsers for characters `(`, `,`, and `)` to create a
parser for tuples. Finally, we can combine that parser with itself a few times
to create a parser for our language.

It is important for me to state that parsers are a very wide field of study;
the theory behind which parser can parse which language is complex. Parser
combinators are not the be-all-end-all of parsers, but they are still a
valuable learning tool, and can be very useful for quickly constructing parsers
for real-world applications. So let's build one from the ground up, starting
with the necessary datatypes.

## Our parser datatype

There is a brilliant quote and mnemonic about parsers that reads as follows;

> A parser for things
>
> is a function from strings
>
> to a list of pairs
>
> of things and strings

Surprisingly, this text contains all the information we need to construct our
parser datatype. Let's construct a type synonym for our parser, line by line:

> A parser for things...

This line implies that our parser is polymorphic in the type of value that it
parses, so let's start out by defining a polymorphic type alias:

```haskell
type Parser t = ...
```

This line is important, because it allows us to easily define parsers for
different kinds of data. In our small tuple language example, for example, we
need parsers for individual characters (`Char`), numbers (`Integer`), tuples
(`(Integer, Integer)`), and lists of tuples (`[(Integer, Integer)]`).

> ...is a function from strings...

Now we know that our parser type is a function type, the argument of which is a
string. Thus, we can expand our type alias as follows:

```haskell
type Parser t = String -> ...
```

This step is perhaps obvious, in the sense that the input to our parser is
always a string.

> ...to a list of pairs...

From this line, it follows that the return type of our parser function is a
list of tuples, although we won't know which types are in that tuple until the
next line:

```haskell
type Parser t = String -> [(..., ...)]
```

The fact that string show up here is perhaps curious at first, but let's
consider what happens when we try to parse "one or more digits" from the
following strings:

 1. `0`
 2. `9185`
 3. `a`

In the first case, the only thing we can really parse is `0`. In the second
case, however, we can parse `9`, `91`, `918`, and `9185`. All of these satisfy
the criterion of being one or more digits. In the third case, we cannot
actually parse anything at all; there is no possibility to meet the parsing
criterion. Storing our parsing results in a lists very elegantly captures the
behaviour we see here; because a list can have any number of elements, we can
model parses which return multiple possibilities, as well as parses which do
not return any parses at all. In a sense, we can view the empty list `[]` as a
parsing error.

> ...of things and strings.

Finally, we find out what is in the tuples we are returning. The first element
is the thing we are parsing, and the second element is another string:

```haskell
type Parser t = String -> [(t, String)]
```

The first part of this tuple models the fact that we are, in fact, reading some
object of type `t`. The second part, the string, is perhaps less obvious. This
models the fact that we must always keep track of what remains of the string
after our parsing is complete. Imagine, for example, that we want to parse "one
or more digits, followed by one or more letters", and our string is `123ab`. As
we discussed before, the first part of this (parsing one or more digits) can
match either `1`, `12`, or `123`. If we then want to parse our letters, we must
do that on whatever _remains_ of the input after we parse our digits. If we
parse `1` as our digits, the remaining string is `23ab`, from which we cannot
parse any letters. If we parse `12`, the remainder is `3ab` which also does not
meet our letter criterion. Only if we parse `123` and leave `ab`, we can satisfy
the second parsing criterion.

We can now sneakily look ahead at how our parsers will _combine_ later. The
result of the parsing one or more digits from `123ab` returns the following
possibilities:

```haskell
[("1", "23ab"), ("12", "3ab"), ("123", "ab")]
```

If we now combine this with our parser for a single letter, all we need to do
is run that parser on the remainder parts of the previous results. In other
words, we run our letter parser on `23ab`, `3ab`, and `ab`, each of which
returns its own list of results:

```haskell
-- From ("1", "23ab")
parse "23ab" = []

-- From ("12", "3ab")
parse "3ab" = []

-- From ("123", "ab")
parse "ab" = [("a", "b"), ("ab", "")]
```

A combined parse can only succeed if the first component manages to produce a
non-empty list of results, _and_ the second component, when applied to the
remaining strings following the first part, also return at least one non-empty
result. The result of the combined parser, in other words, is the product of
each initial result with the corresponding subsequent parsing result. In this
case, we will join our string using the concatenation operator, although this
is not necessary in general. In the above example, we find the following
results for our combined parser.

```haskell
[("123" ++ "a", "b"), ("123" ++ "ab", "")]
```

There are two remaining issues with our parser type. The first problem is that
we will want to define class instances for our type, which is (by default) not
possible for type synonyms. For this reason, we will want to turn our type into
a "real", non-synonym type. We we can do so by declaring it as a `newtype` with
its own constructor:

```haskell
newtype Parser t = Parser (String -> [(t, String)])
```

Finally, let's do away with this idea that parsers must always operate on
strings of characters, and make the string type generic too. Remembering that
`String` is just a synonym for `[Char]`, we can make our type polymorphic as
follows:

```haskell
newtype Parser' s t = Parser ([s] -> [(t, [s])])
type Parser = Parser' Char
```

In this case, we'll call our generic parser type `Parser'` (note the tick), and
we will partially specialize our type using `Char` to represent the common case
of parsing strings, and we will call this type `Parser`. Notice that `Parser`
is exactly the same type we had before.

## Our first parser

Let's now write our first parser. The function of this parser will be to read a
single item -- regardless of what it is -- from our input string. We'll call
this parser `read1`. Its type will be the following:

```haskell
read1 :: Parser' a a
```

Depending on how familiar you are with writing Haskell, this type signature
alone might seem intimidating. Specifically, when I first saw a type signature
like this I wondered how this could work without the type signature being a
function. Indeed, this is not -- at first sight -- a function type signature,
because there's no `->` to be found. If you're confused by this, please try to
remember that we defined `Parser'` to contain a function earlier! We _are_
returning a function here, but it's wrapped in the `Parser'` type.

Let's go ahead and add a definition for this `read1` parser now. Remember that
we build a `Parser'` term by taking a function of type `[s] -> [(t, [s])]` and
wrapping that in the `Parser` constructor. Please keep in mind that `Parser` is
both a _type constructor_ and a _value constructor_, and we must be cognisant
of when we use which. In this case, we are using it in the universe of terms,
and as such it is a value constructor. Let's see if we can simplify the
problem:

```haskell
read1 :: Parser' a a
read1 = Parser f
    where
        f :: [a] -> [(a, [a])]
        f = ...
```

I am contractually obliged to tell you that this code is not quite correct;
Haskell will interpret the `a` type in the signatures for `read1` and `f` as
independent free types, which is somewhat annoying. We'll get rid of the type
signature for `f` in a second, it's just here to provide us with some handholds
to get started. You can also turn on the `ScopedTypeVariables` language
extension to make this work.

Now, let's think of what the function `f` (which we're wrapping in a `Parser`
constructor) should do. Whenever we design a parser like this, we should think
of when we fail, and what is left when we succeed. In this particular case, we
really need to think of only two scenarios: either the input list is non-empty,
in which case we can read the first element and return the rest of the string
as the remainder, or the string is empty and we fail to parse anything. This
allows us to elegantly complete our definition for `read1`:

```haskell
read1 :: Parser' a a
read1 = Parser f
    where
        f (x:xs) = [(x, xs)]
        f []     = []
```

## Running our first parser

So far, our singular parser is... Not particularly useful. We've wrapped a
function in some constructor, but we don't have a way to actually parse any
text with it. Let's therefore, write a function which we will use to actually
_parse_ things. In othSomething went wrong, but don’t fret — let’s give it
another shot. er words, once we've constructed our elaborate parsers through
combining them, this function will run them. It may surprise you, at first,
that we will only ever need this single function to run any parser we can come
up with.

First, let's come up with the type signature for our `parse` function.
Obviously, we will need to pass it some `Parser'` object; we can look at this
as the "guidebook" on how to parse a particular thing. Secondly, we will need
something to run the parser on. Remember that `Parser' s t` operates on a list
type `[s]`, and as such we will take that as our input. Finally, we will want
to return an object of type `t`: the thing that we have actually parsed. This
gives us the following type signature:

```haskell
parse :: Parser' s t -> [s] -> t
```

And now, something beautiful starts to happen, something which I only really
ever experience in Haskell: the type system will essentially start to write our
code for us. As we look at the types of the things we have we will find there
are very few sensible ways in which they fit together. Let's first remember
that our parsing function is wrapped in a constructor, so let's pattern match
to unwrap it. I'll do my best to annotate the next few code segments with the
type signatures for the different bindings (mind you, the syntax here is
mangled a little to guide you).

```haskell
parse :: Parser' s t -> [s] -> t
parse (Parser f) s = ...
    where
        s :: [s]
        f :: [s] -> [(t, [s])]
```

At this point, the only sensible way to do is to apply the (now unwrapped)
function `f` to the input string `s`, so we get the following:

```haskell
parse :: Parser' s t -> [s] -> t
parse (Parser f) s = ...
    where
        s :: [s]
        f :: [s] -> [(t, [s])]
        r :: [(t, [s])]
        r = f s
```

We now have this binding `r` of type `[(t, [s])]`, and we are looking for a
value of type `t` to return. Remember that the reason `r` is a list type is to
model the idea that we want to support multiple possible parses. However, each
of these parses will always satisfy all of our requirements (at least, if we
correctly implement the rest of our parsing library). As such, it doesn't
really matter which element of this list we use; we'll just grab the first one:

```haskell
parse :: Parser' s t -> [s] -> t
parse (Parser f) s = ...
    where
        s :: [s]
        f :: [s] -> [(t, [s])]
        r :: [(t, [s])]
        r = f s
        u :: (t, [s])
        u = head r
```

Now we have a tuple `u` of type `(t, [s])`. Remember that at this point we've
already done our parsing -- we're really just post-processing our results --,
and as such we do not need the remainder `[s]` anymore, so we can simply grab
the first element and return it:

```haskell
parse :: Parser' s t -> [s] -> t
parse (Parser f) s = v
    where
        s :: [s]
        f :: [s] -> [(t, [s])]
        r :: [(t, [s])]
        r = f s
        u :: (t, [s])
        u = head r
        v :: t
        v = fst v
```

Now all that remains is to clean this up, which we can do as follows:

```haskell
parse :: Parser' s t -> [s] -> t
parse (Parser f) s = fst . head . f $ s
```

Let's see if we can now run our first parser. Remembering that our `read1`
parser takes the first element and returns it, think of what happens when we
run the `read1` parser on the string `"hello world"`, and what happens when we
run it on `[True, False, False]`?

```haskell
parse read1 "hello world" = 'h'
parse read1 [True, False, False] = True
```

Great! While we have, in essence, built a glorified `head` function, we have
made a big first step towards building our parser library. Let's take a look,
now, at how we can construct more complex parsers.

## Naively combining parsers

As we discussed before, we build more complex parsers by combining simpler
parsers. Let's look at how we can do this naively, and then we can look at how
to do this more elegantly. We'll construct a new parser, `read2`, which reads
_two_ characters from the input. It's type and definition will look like this:

```haskell
read2 :: Parser' a [a]
read2 = read1 `andThen` read1
```

Now we need to define the `andThen` function, which has the following type:

```haskell
andThen :: Parser' a a -> Parser' a a -> Parser' a [a]
```

We'll need to remember a few things in order to write this function. First,
let's remember that the two `Parser'` objects we have both wrap some a
function. Secondly, we should remember that we are returning a new `Parser'`
object, which we will need to construct with the `Parser` constructor. This
gives us the following to fit in:

```haskell
andThen :: Parser' a a -> Parser' a a -> Parser' a [a]
andThen (Parser f) (Parser g) = Parser h
    where
        f, g :: [a] -> [(a, [a])]
        h :: [a] -> [([a], [a])]
```

Now, we will need to write our function `h`. Remember that each of our parsing
functions `f` and `g` return whatever they parse, as well as the remaining
string, and we need to pass the remainder of our first function to the second
function. Let's nest some additional bindings by applying our first parsing
function to the input of our new parser to clarify:

```haskell
andThen :: Parser' a a -> Parser' a a -> Parser' a [a]
andThen (Parser f) (Parser g) = Parser h
    where
        f, g :: [a] -> [(a, [a])]
        h :: [a] -> [([a], [a])]
        h s = ...
            where
                r :: [(a, [a])]
                r = f s
```

Remember that we need to apply our second parsing function `g` to _every_
remainder returned by `f`, and then we need to combine those results. In this
case, we want to put the two individual elements into a list. Also, the
remainder returned by our new combined parser needs to be whatever is returned
by the second parser (remembering that whatever was required by the first
parser has already been consumed). We can do this using a list comprehension:

```haskell
andThen :: Parser' a a -> Parser' a a -> Parser' a [a]
andThen (Parser f) (Parser g) = Parser h
    where
        f, g :: [a] -> [(a, [a])]
        h :: [a] -> [([a], [a])]
        h s = [([i1, i2], xss) | (i1, xs) <- r, (i2, xss) <- g xs]
            where
                r :: [(a, [a])]
                r = f s
```

Let's simplify this into a simpler expression:

```haskell
andThen :: Parser' a a -> Parser' a a -> Parser' a [a]
andThen (Parser f) (Parser g) = Parser h
    where
        h s = [([i1, i2], xss) | (i1, xs) <- f s, (i2, xss) <- g xs]
```

Understanding what is happening here is _critical_ to understanding parser
combinators. Please take the time to read over this function several times to
understand what's happening. We're applying our _first_ parser function `f` to
the initial input `s`, creating a set of results `(i1, xs)`, where `xs` is each
of the remainders. For each results of `f`, we run `g` on the remainder
produced by that result of `f`, which in turn produces another set of results
`(i2, xss)`, which represent the thing parsed by the second function, as well
as the remained after _both_ parsers have been run.

Let's see how this new combined parser works on the inputs that we used
earlier:

```haskell
parse read2 "hello world" = "he"
parse read2 [True, False, False] = [True, False]
```

There are (at least) two glaring issues with this approach. The first one is
that we cannot control the way in which our two elements are combined. For
example, what if we want to return a _tuple_ of our two elements instead of our
list? What if we want to return a much more intricate data structure? Let's see
if we can abstract the combination function:

```haskell
andThenWith
    :: (a -> b -> c)
    -> Parser' s a
    -> Parser' s b
    -> Parser' s c

read2 :: Parser' a [a]
read2 = andThenWith (\x y -> [x, y]) read1 read21
```

That's quite an intimidating type signature! Thankfully, in this case we can
fill it out this rather easily. We simply need to modify the list comprehension
we used in our initial refinition of `andThen` to look as follows:

```haskell
andThenWith
    :: (a -> b -> c)
    -> Parser' s a
    -> Parser' s b
    -> Parser' s c
andThenWith c (Parser f) (Parser g) = Parser h
    where
        h s = [(c i1 i2, xss) | (i1, xs) <- f s, (i2, xss) <- g xs]
```

Now, we can define some more interesting parser combinations:

```haskell
read2Tuple :: Parser' a (a, a)
read2Tuple = andThenWith (,) read1 read1

read2IgnoreFirst :: Parser' a a
read2IgnoreFirst = andThenWith (\_ x -> x) read1 read1

read2Reverse :: Parser' a [a]
read2Reverse = andThenWith (\x y -> [y, x]) read1 read1

read2ButRepeatTheFirstFiveTimes :: Parser' a [a]
read2ButRepeatTheFirstFiveTimes = andThenWith
                                  (\x y = [x, x, x, x, x, y])
                                  read1 read1
```

I'm sure you get the idea. The second problem, which is far more glaring, is
that there is very little structure in the way we have written so far. Writing
Haskell if very often about finding abstracts structures in the code you're
writing, and then leveraging the work that other people have already done with
that structure. It turns out that our parser type has a lot of such structure,
including some structure that will allow us to combine our parsers much more
easily. Let's see what kind of structure we can discover in our parser type. By
the end of this post, we will have replaced all the combination code we just
mentioned by more abstract structures, which will make it much easier to work
with.

## Parsers as functors

First, we will start by capturing the functorial nature of our parsing objects.
I will include a brief explanation of what functors are first, so feel free to
skip past that part if you're already familiar with functors.

### An introduction to functors

In my limited experience with Haskell, functors are perhaps _the_ most
important concept to understand. As a result, there are thousands of different
explanations of how functors work on blogs all over the internet. Particularly
infamous is the burrito metaphor, so I will try to stay away from that. At it's
core, a functor is a _type constructor_ that allows us to lift functions on
"naked" types into its constructed types "for free".

If you're unsure what a type constructor is, it's like a function that operates
on the _type level_ and turns one type into another. In Haskell, `Integer` is a
type. We also have the type `Maybe Integer`. In this case, `Maybe` is a type
constructor: it takes a type (in this case, `Integer`) and returns a new type
(in this case, `Maybe Integer`). The key insight here is that `Maybe` by itself
is _not_ a type; we cannot have a value of type `Maybe`. In a more formal
sense, we say that the _kind_ of `Maybe` is `* -> * -> *`, where `*` is the
kind of all types.

You can interpret the functorial nature of `Maybe` in two ways. Firstly, we can
think of this as the ability to _modify_ the contents of a `Maybe t` value as
though it was just a `t`. The other way of looking at it is to think that we
can transform any function `t -> u` to a function `Maybe t -> Maybe u` for
free. Personally, I think the second interpretation is a little more useful but
either one works. Consider these two equivalent type signatures of `fmap`,
which is the crucial function to defining a functor:

```haskell
fmap :: (a -> b) -> f a -> f b
fmap :: (a -> b) -> (f a -> f b)
```
