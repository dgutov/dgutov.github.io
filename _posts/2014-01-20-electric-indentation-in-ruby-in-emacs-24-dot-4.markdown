---
layout: post
title: "Electric Indentation in Ruby in Emacs 24.4"
date: 2014-01-20 03:14:40 +0200
comments: true
categories: blog
---

## Intro

Although `electric-indent-mode` will be
[on by default](http://emacsredux.com/blog/2014/01/19/a-peek-at-emacs-24-dot-4-auto-indentation-by-default/)
in Emacs 24.4, most of the time it is used only in the most
straightforward way possible: the major mode adds some characters to
`electric-indent-chars`, and after the user types one of them, the
current line gets reindented.

This limits us to electric indentation only after specific characters,
not keywords, and without regard to the context.

However, `electric-indent-mode` allows a more advanced behavior, and
for that one needs to set `electric-indent-functions`.

So far, only two modes bundled with Emacs use it: `ruby-mode` and
`perl-mode`, and the latter only to disable electric indentation
whenever point is not at the end of the line.

## Motivation

But when we write a function, we can look at the full symbol before
point, see if it's a keyword, and if it starts at indentation, or if
it was a keyword before we typed the last character. We can look at
the character after point and see whether we just turned a
continuation method call into a straight metod call, which does not
need additional indentation.

## Examples

`|` denotes the cursor position.

```ruby
# before
if foo
  bar
  els|
end

# after
if foo
  bar
else|
end
```

```ruby
# before
if foo
end|

# after
if foo
  ends|
```

```ruby
# before
foo
|

# after
foo
  .|
```

```ruby
# before
foo
  |.bar

# after
foo
t|.bar
```

The above list is based on my personal preference, so please let me
know how it works for you.

Hopefully, this general behavior will also spread to other major
modes.
