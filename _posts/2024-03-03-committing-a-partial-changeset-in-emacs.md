---
layout: post
title: Committing a partial changeset in Emacs
date: 2024-03-03 22:15 +0200
comments: true
categories: blog
---

One of the frequent questions for the built-in Version Control support
in Emacs has been the ability to commit only a part of the changes in
a file or several.

In the command-line, it's simple (though not always easy): you call
<kbd>git add -p</kbd> and iterate through the whole repository diff
with <kbd>y</kbd>/<kbd>n</kbd>/etc. [Magit](https://magit.vc/) also
has an interface for staging and committing from the staging area, but
that only works for Git and requires you to switch to its UI first.

## Using VC

One of the less noticed additions in Emacs 29 is the ability to
[create a commit from a diff buffer](https://debbugs.gnu.org/52349).
And since `diff-mode` provides functionality for editing the diff, you
can drop the unwanted pieces first (they stay saved on disk) and
commit the amended changeset.

Here's how it works: you press <kbd>C-x v =</kbd> or <kbd>C-x v
D</kbd> to show the diff for the current file, or the whole
repository, edit the diff using commands such as `diff-hunk-kill`
(<kbd>M-k</kbd>), `diff-split-hunk` (<kbd>C-c C-s</kbd>), and then
type <kbd>C-x v v</kbd> in the same buffer to proceed to the familiar
step "Edit commit message".

I also recommend customizing `diff-default-read-only` to `t` so that
the short key combinations work right away in `diff-mode`, including
<kbd>n</kbd> and <kbd>p</kbd> for hunk navigation and <kbd>k</kbd> for
removing hunks. Otherwise those only work after switching to
read-only-mode (<kbd>C-x C-q</kbd>)

Anyway, you enter the commit message and press <kbd>C-c C-c</kbd> to
create the commit. This works not only with Git, but also with
Mercurial repositories, and the fallback implementation endeavors to
support other VC systems as well (such as Bazaar and SVN), though
using a slower method. The latter is not as well-tested (the latest
bug report was regarding an incompatible command-line flags in
OpenBSD's `patch`), so more feedback welcome.

{% video https://github.com/dgutov/dgutov.github.io/assets/271877/a491495d-e411-4296-8f6b-720b54babf7a#.webm 580 540 %}

## Using Diff-HL

The workflow described above doesn't use the staging area, and if you
are using Magit, you might prefer a simply faster way to stage hunks.

By popular demand, the new additions to the
[diff-hl](https://github.com/dgutov/diff-hl/) package are the commands
`diff-hl-stage-current-hunk` and, more recently, `diff-hl-stage-dwim`.

Either type <kbd>C-x v S</kbd> to stage the hunk at or above point, or
prepend it with <kbd>C-u</kbd> to similarly pop up a buffer where you
can edit the changeset to stage. Then do the commit with Magit or the
command line.
