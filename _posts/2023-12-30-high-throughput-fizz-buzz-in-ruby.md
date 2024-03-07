---
layout: post
title: High throughput Fizz Buzz in Ruby
date: 2023-12-30 12:34 +0200
categories: blog
---
Stumbled on a StackExchange competition of sorts, where the [top
answer](https://codegolf.stackexchange.com/a/236630/120701) is in
hand-rolled assembler that reimplements libc functions in a
specialized way to avoid copying as much as possible, uses special
memory alignment for storing numbers, and a bytecode generator and
interpreter for the main loop. Reading it is a nice deep rabbit hole.

There was only one solution in Ruby, however, which seemed unfair. So
[I tweaked it a little](https://codegolf.stackexchange.com/a/268828/120701).

It doesn't do too bad in the overall comparison (see the summary
comparison table [inside the
question](https://codegolf.stackexchange.com/q/215216/120701)), so I
think it demonstrates how the IO performance is its own challenge, and
how runtime adaptation in a dynamic language can succinctly overtake a
naive solution in a statically compiled one, and get within an order
of a magnitude of an average solution in a decently fast language.

[Also, Ractors](https://codegolf.stackexchange.com/a/268828/120701).
