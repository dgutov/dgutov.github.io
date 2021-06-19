---
layout: post
title: Debugging a syntax error in js2-mode
date: 2021-06-19 03:27 +0300
comments: true
categories: blog
---

I haven't been writing much JavaScript code lately, so my personal
motivation to work on [js2-mode](https://github.com/mooz/js2-mode/)
has waned over the years, but it's a cool package with some dedicated
user base. I'm sure there are people who have wanted to contribute,
but didn't know where to start.

Below is the story of debugging and fixing a certain bug which took a
bit more effort than expected. It can probably serve as a primer both
for hacking `js2-mode` and Emacs Lisp in general. Some pre-existing
knowledge of Lisp is required, though.

## The Problem

[The bug](https://github.com/mooz/js2-mode/issues/480) related to the
parsing or trailing commas in arrow functions, which is apparently
encouraged in some coding styles. This worked in normal functions
(the feature was added several years prior):

```javascript
function foo(a, b,) {
  console.log(a, b);
}
```

But not in arrow functions:

```javascript
(a, b,) => { console.log(a, b) };
```

An educated guess from one of the reporters was to check
`js2-parse-function-params` and see whether it's doing something
wrong. This function parses parameters for all kinds of functions
(expression, statement or arrow).

## Late introduction

`js2-mode` uses a recursive-descent parser written in Emacs Lisp,
originally ported from [Mozilla
Rhino](https://github.com/mozilla/rhino) by [Steve
Yegge](https://steve-yegge.blogspot.com/2008/03/js2-mode-new-javascript-mode-for-emacs.html).

So most of its code falls into one of 3 categories:

* Type definitions for the values representing the syntax tree nodes, using `cl-defstruct`.

* The token stream and the tokenizer. Main entry point is `js2-get-token`.

* The parser code, which is a bunch of functions that read the tokens
  one by one and call each other in a recursive fashion, according to
  the JavaScript grammar, while employing a couple of tricks to
  resolve the ambiguities (when you can't tell in advance what kind of
  node you are looking at). And it constructs the syntax tree based on
  the tokens encountered.

`js2-parse-function-params` is one of such functions, it helps reach
the list of function parameters and organizes them into a list, which
is then attached to the function node.

## Debugging

So it's a good place to start[^1]. Let's switch to `*scratch*`, `M-x
js2-mode` in there and paste the problematic snippet of code. It
flashes in red: syntax errors.

We switch to another window and visit `js2-mode.el` in it. Navigate to
`js2-parse-function-params`'s definition and instrument it to be
debugged with `C-u C-M-x`. The echo area briefly flashes:

> Edebug: js2-parse-function-params

Switch to `*scratch*` and type space somewhere at the end, forcing the
buffer to be re-parsed. Emacs stops at the beginning of the
instrumented function.

We step through it, to the end, by tapping `SPC`, noting the return
values displayed in the echo area.

Things look wrong, starting from the return value of
`(js2-peek-token)` at the beginning of the loop, followed by
`js2-must-match-name` failing later, and `(js2-create-name-node)`
creating a node with some odd `:name` attribute.

The first return value was the best hint, though. It returned 165
(token types are numbers), which, if you `C-s` through the file, leads
to this line:

```scheme
(defvar js2-ARROW 165)         ; function arrow (=>)
```

That means the tokenizer says that the next token after the current is
the arrow (`=>`) already. We can double check that by seeing the value
of `js2-ts-cursor` (by typing `e` while debugging and typing the
variable's name in the prompt). Its value is 12, the position right
after the arrow token.

But when we are parsing some construct, we
really expect to be near its beginning (either at the first token, or
before it). How come we are not there?

Let's look at the list of errors. Type `M-x js2-display-error-list`,
and it brings up a bunch of warnings, as well as these three errors:

```
line 1: missing ) after formal parameters
line 1: missing formal parameter
line 1: missing ) in parenthetical
```

Searching for the first two brings us back to
`js2-parse-function-params`. These errors are to be expected.

The last one is more interesting. The error message key to search for
is `"msg.no.paren"`, and it appears in 3 functions.

Try instrumenting them all. Only one of them is entered by the code
parsing this example: `js2-parse-paren-expr-or-generator-comp`.

What does it do? It checks whether the current token is followed by
`for`, or a right paren right away, and when it doesn't, it calls
`js2-parse-expr` and then checks that the said expression is followed
by `js2-RP`, which is the "right paren" token. That fails to happen
in our case.

Look at `js2-parse-expr`. It calls `js2-parse-assign-expr` in a
loop, as long as it finds new `js2-COMMA` tokens.

We instrument `js2-parse-expr`. Stepping through it, we can see that it
ends up calling `(js2-parse-assign-expr)` twice. The first call
returns a `js2-name-node`, totally expected. The second call, inside
`(setq right (js2-parse-assign-expr)`, returns a `js2-function-node`
(the return value printed in the echo area looks complex, but we can
make sure by evaluating `(aref right 0)`). That's a surprise.

To see how this happens, let's go down the rabbit hole. First of all,
reset all instrumentation by calling `M-x eval-buffer` in `js2-mode.el`.
Then let's simplify the code so the parser does less work for us to follow:

```javascript
(a,) => { };
```

Then instrument `js2-parse-expr` with `C-u C-M-x`, make an edit in the
code buffer and step through the source until the cursor is after
`(setq right `. Press `i` to jump in.

Step through and jump inside every successive call to functions named
`js2-parse-*`. Meaning, `js2-parse-assign-expr`, then to
`js2-parse-cond-expr`, and so on. When we reach
`js2-parse-primary-expr`, it finishes with reporting a syntax error
and creating a `js2-error-node`. The thing to note here is it
"consumes" the current token. The closing paren has just been parsed
as an error node.

As we continue with stepping through the code with `SPC`, we can see
the execution bubble up to `js2-parse-assign-expr`. In there, we see
`(= tt js2-ARROW)` condition match, token stream rewound back to the
position before the last node parsed, and then parsed again as
function.

## Erase and Rewind

What's that about rewinding?

The ability to save and restore the current stream state was
added for the implementation of the arrow functions' support.

When the parameter list is parsed first, there is no good way to know
in advance that it belongs to an arrow function, and it's not just a
parenthesized expression. So the parens are first parsed as an
expression, and then if it's followed by the arrow token, the stream
is rewound and the code is parsed as a function.

For all of this to work, any valid parameters list of an arrow
function has to be parse-able as a parenthesized expession, too.

At some point you might have been wondering: if the parameter list was
not parsed correctly, and it was not followed by the arrow token, how
come `js2-parse-function-params` was called at all? Hopefully the
previous section answered that question [^2].

## Solution

One possible direction would be to change `js2-parse-primary-expr` not
to consume the current token. Then the inner `js2-parse-assign-expr`
call wouldn't match `js2-ARROW` as next token, `js2-parse-expr` would
find the closing paren fine, and the call to `js2-parse-assign-expr`
up the stack which we never actually stepped through, would find the
arrow token in its proper place and parse the arrow function fine.

Unfortunately, that's kind of dangerous, that approach can lead to
infinite looping. Error nodes should consume tokens to ensure that the
parser moves forward.

The solution I ended with is a simplified version of Mozilla's code
from Firefox. When a feature is already implemented in there, it's
often handy to refer to its code (even if it looks much busier than
the elegant Lisp we are currently editing). The general structure is
similar enough, the hierarchy of calls in particular.

The function in question is
[GeneralParser::expr](https://github.com/mozilla/gecko-dev/blob/b2b0f5cd62774d1682a3bcf935ee739ca9d8c30e/js/src/frontend/Parser.cpp#L9218).
We can see that rather than allow just any kind of errors inside, it
handles two special cases (trailing comma and triple dot operator),
and only when within a context where it's possible that the curent
node is a function parameter list.

The [change I checked
in](https://github.com/mooz/js2-mode/commit/5e9515dff26ed2fa382c97b0b67cdd370f7859f4)
is much shorter because we don't really need to worry about (not)
reporting syntax errors: whenever we rewind the token stream to a
previous position, we change `js2-parsed-errors` to its previous value
as well (see the end of `js2-parse-assign-expr`).

Performance impact[^3] from this change was very modest (if noticeable
at all), so it looks like a success.

Thanks for reading.

[^1]: If you want to follow the scenario step-by-step, you need to check out the version of code that was used at the time, Git commit `b891ede`.

[^2]: To recap: what was recognized as the "parameter list" expression wasn't followed by the arrow token, but the "error node" inside it was. So the arrow function was parsed from that position, and `js2-parse-function-params` was called then.

[^3]: To measure it, you can visit some large-ish file and evaluate `(benchmark 10 '(js2-reparse t))`. Then change the implementation, `M-x eval-buffer` and run the benchmark again. Do this a few times, to get the feel for the spread.
