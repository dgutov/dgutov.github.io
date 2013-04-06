---
layout: post
title: "Emacs 24.3's Killer Feature: Eager Macro-Expansion"
date: 2013-04-07 00:03
comments: true
categories:
---

Looks like this change has gone largely unnoticed, aside from occasional bug
reports when it failed and emitted a warning.

Meanwhile, the speed-up it provides for uncompiled code ranges from nice to
amazing, depending on the amount and complexity of macros used.

## Intro

I first [noticed](http://debbugs.gnu.org/cgi/bugreport.cgi?bug=13605) the
difference when benchmarking an [mmm-mode](https://github.com/purcell/mmm-mode)
function that calls `syntax-propertize-function` from different major modes,
including `ruby-mode`.

`ruby-syntax-propertize-function`, like most of the similar functions, uses
`syntax-propertize-rules`, a distinctly complex macro. The difference between
interpreted and compiled code was orders of magnitude, and it was especially
noticeable in `mmm-syntax-propertize-function`, because the ERB code example I
usually use for performance testing has ~200 ERB regions, so that's the amount
of times `ruby-syntax-propertize-function` was called.

## Some numbers

For a more practical example, let's measure the time
[js2-mode](https://github.com/mooz/js2-mode/) parser takes to process a large
source file.
All ~1500 lines of [uncompressed Backbone.js](http://backbonejs.org/backbone.js).

`js2-mode` has always been notoriously slow in interpreted mode, due to the
heavy use of `defstruct` facility and other macros from the `cl` package.

   Version | Interpreted (time, s) | Compiled (time, s)
-----------|-----------------------|-------------------
Emacs 24.2 |                6.4251 |             0.3025
Emacs 24.3 |                0.5026 |             0.2524

So, compiled code became a bit faster. Not critical, but nice.

Interpreted code became *a lot* faster, losing to the compiled code only by the
factor of 2.

This is huge, it means that we can drop the strict recommendation to compile the
package when installing manually, for Emacs 24.3 and later. I'm in no hurry to
change the doc, but the amount of dissatisfied keyboard jockeys who routinely
skip the documentation will go down, at least on this subject.

It's especially nice for me personally: having to recompile the code after
making some changes has always been a pain. Authors of other Elisp packages and
users with a lot of code in their init files should also see the benefit.

## More details

Not having studied the innards of `bytecomp.el` in detail, I'll stick to what we
can glean from experiment.

A good way to see what some function really does is fire up the Emacs Lisp REPL
(`M-x ielm`) and there evaluate `(symbol-function 'foo)`, where `foo` is the
function in question.

As an aside, this is also a good way to find out what some macro like
`define-minor-mode` does without studying it in detail: just look at the body of
the resulting `-mode` function.

Take this definition:

```scheme
(defun foo ()
  (when t
    (dotimes (i 10))))
```

With Emacs 24.3, you can (sometimes unwittingly) avoid the eager macro-expansion
by using `eval-last-sexp` (`C-x C-e`) instead of `eval-defun` (`C-M-x`) or
`eval-buffer` (no default binding). So we can try it both ways.

Without eager expansion, the body looks very familiar:

```scheme
ELISP> (symbol-function 'foo)
(lambda nil
  (when t
    (dotimes
        (i 10))))

ELISP> (js2-time (dotimes (i 1000) (foo)))
0.0367
```

With eager expansion, we can see that it has been reduced to basic forms:

```scheme
ELISP> (symbol-function 'foo)
(lambda nil
  (if t
      (progn
        (progn
          (let
              ((--dotimes-limit-- 10)
               (i 0))
            (while
                (< i --dotimes-limit--)
              (setq i
                    (1+ i))))))))

ELISP> (js2-time (dotimes (i 1000) (foo)))
0.0086
```

The numbers fluctuate heavily with repeated invocations, but the second version
is always several times faster. The "familiar" version has to expand the macros
each time the function body is evaluated, which drags the performance down
significantly.
