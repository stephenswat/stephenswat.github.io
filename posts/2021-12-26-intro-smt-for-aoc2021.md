---
title: Introduction to Satisfiability Modulo Theories for Advent of Code 2021
date: 2021-12-26
author: Stephen Nicholas Swatman
---

The twenty-forth challenge of this year's _Advent of Code_ was both an
interesting one and a controversial one. In this puzzle, we are given some
machine code for a simple six-instruction, four-register computer. The
objective, then, is to compute the maximal and minimal inputs that are accepted
by the given machine code (acceptance, in this case, is implied by writing a
certain value to a certain register at the end of the computation).

This problem frustrated me. Not because it was particularly difficult, but
because the "intended" solution requires the participant to, in essence,
manually disassemble the machine code in order to decipher what it does in a
more abstract sense. This, to me, is somewhat contrary to the core idea of
Advent of Code: for every other problem, the puzzles promote coming up with a
general solution. In fact, the Advent of Code website very often provides some
example inputs to test your program on, after which your sufficiently general
solution _should_ be able to provide a correct answer to your own unique input.
As far as I am concerned, day 24 of this year's Advent of Code calendar goes,
frustratingly, against that philosophy. The good news is that there does exist
a general solution to this problem; the bad news is that it may be difficult to
find -- if you don't know where to look.

When you're solving a problem -- be it for Advent of Code, for work, or for any
other reason --, the most frustrating thing that can happen is being unable to
express a problem in a way that allows you to find a solution. In my view, this
is _the_ main point of formal education: to acquire sufficiently broad
knowledge about a field that you can grasp at certain concepts you know in
order to learn more about concepts you don't. For example, if I were confronted
with a shortest path problem (which are usually in no short supply in Advent of
Code), I would know to reach for Dijkstra's algorithm. Someone who might not
know the specific algorithm but is familiar with graphs might know to search
for "shortest path algorithm" and find a similar approach. But what if you
don't know what a graph is at all? How do you even _start thinking_ about such
a problem in a productive way?

To Eric's credit, Advent of Code is usually quite good at providing some subtle
hints about where to start. In addition, there is a broad community around
Avent of Code which seems friendly and welcoming, and where people new to
computing can go for help with such matters. Unfortunately (or fortunately for
me because I get to write a blog post about it), I have seen very little
discourse about what I would argue is the "proper" way to solve this problem.
This is understandable, because the method is relatively obscure, but also
immensely powerful, and as such I would like to dedicate this blog post to
introducing you to it. Like you might teach someone looking to solve a shortest
path about introductory graph theory and Dijkstra's algorithm, I would like to
introduce you to _satisfiability modulo theories_ (SMT) problems. At the end of
this post, I hope to have conveyed to you what SMT problems are, how to use SMT
solvers to solve this Advent of Code problem, and to have given you some idea
of when you can reach for this approach in your other programming work.

## Boolean satisfiability problems

Let's imagine, for the sake of the Christmas spirit, that you are organizing an
office Christmas party. This, of course, is an exercise in office politics, and
you will need to consider all kinds of inter-personal relationships. Let's say,
for example, that:

1. Alice (`A`) and Bob (`B`) dislike each other, and as such you can only
   invite one of them; if you invite both it will be terribly awkward.
2. Alice (`A`) and Christine (`C`) are a couple, and if you invite either of
   them you should invite the other.
3. Christine (`C`) is Dave's (`D`) boss, and if you invite Christine, Dave will
   want to be there to make a good impression.
4. Bob (`B`) and Dave (`D`) both love Emily's (`E`) chocolate brownies, and if
   you invite Emily to the office party, you should not invite both Bob and
   Dave lest they cause a big fight over who gets the last brownie and ruin the
   office party.

As you begin to pull out your hair trying to solve this difficult problem, you
may wish to cancel the office party all together, so let's add a fifth clause:

5. There must be at least one person (other than you) attending the party.

What we have constructed here is a so-called _boolean satisfiability problem_.
If we treat each potential guest as a boolean-valued variable (where &top;
represents them attending the party, and &bottom; denotes them not being
invited), we can describe all the office politics using logical operators. I've
given each person a one-letter variable, and we can find the following
corresponding boolean predicates:

1. If Alice attends, then Bob should not attend. Similarly, if Bob attends,
   Alice should not attend. Thus, we add the constraint: `(A ⇒ ¬B) ∧ (B ⇒ ¬A)`,
   which can be more simply written as `¬A ∨ ¬B`.
2. If Alice attends, Christine attends, and if Christine attends, then Alice
   should attend. In other words, `(A ⇒ C) ∧ (C ⇒ A)`.
3. If Chrisine attends, then Dave should attend. Therefore, `C ⇒ D`.
4. We can use a similar rule that we used for Alice and Bob to obtain a boolean
   expression of this rule: `E ⇒ (¬B ∨ ¬D)`.
5. This one is perhaps the simplest; as we want at least one person to attend
   the party, we can add the constraint: `A ∨ B ∨ C ∨ D ∨ E`.

Since all of these clauses must be true, we can join them with logical
conjunctions (_ands_) to get the following logical expression:

`(¬A ∨ ¬B) ∧ (A ⇒ C) ∧ (C ⇒ A) ∧ (C ⇒ D) ∧ (E ⇒ (¬B ∨ ¬D))
∧ (A ∨ B ∨ C ∨ D ∨ E)`

In this case, we have five potential attendees, each of which can either attend
or not. As such, we have $2^5 = 32$ possible combinations, which we can faily
easily brute force. Here is a simple and inelegant Python script which prints
the different satisfying combinations:

```python
for A in [True, False]:
    for B in [True, False]:
        for C in [True, False]:
            for D in [True, False]:
                for E in [True, False]:
                    c1 = (not A) or (not B)
                    c2 = (C if A else True)
                    c3 = (A if C else True)
                    c4 = (D if C else True)
                    c5 = (((not B) or (not D)) if E else True)
                    c6 = (A or B or C or D or E)
                    if c1 and c2 and c3 and c4 and c5 and c6:
                        print (A, B, C, D, E)
```

We find that we can come up with eight valid sets of guests:

1. Alice, Christine, Dave, and Emily
2. Alice, Christine, and Dave
3. Bob and Dave
4. Bob and Emily
5. Only Bob
6. Dave and Emily
7. Only Dave
8. Only Emily

We have just solved a _boolean satisfiability problem_, or a _SAT_ problem:
given some expression of boolean-valued variables, is there some configuration
of those variables (which is to say, some combination of them being true or
false) for which the entire expression holds true? In our office party example,
we were able to find some, and we were able to find them quite easily. However,
this problem is not easy in general. Satisfiability problems are so-called _NP
complete_ problems. I won't go into detail about what that means exactly, but
it is for now sufficient to say that these problems do not scale well. If our
company had eighty employees, there would be more possible combinations than
there are grains of sand in the Sahara desert; our naive Python solution would
not hold water -- or sand -- against that many combinations.

Still, with some trickery, we can solve satisfiability problems that are much,
much larger than this. People who are much smarter than me have spent lots of
time thinking of ways to cleverly tackle satisfiability problems, and they have
come up with loads of good algorithms and powerful heurstics for this problem.
Again, how this works in practice is a little outside the scope of this post,
but we can look at an example from the office party above. If you look closely,
you will see that Alice only ever attends if Christine attends, and vice versa.
Thus, the value of `A` in a satisfying answer is always equal to the value of
`C`. That means that we only need to consider one of these variables in our
combinatorics, driving the number of combinations down to only $2^4 = 16$.
Modern dedicated SAT solving programs can handle problems with thousands upon
thousands of variables and return a result in a reasonable amount of time,
which allows us to handle many real-world scenarios in which satisfiability is
important.

## Satisfiability modulo theories

## Advent of Code 2021 day 24 as an SMT problem

## Z3

## 
