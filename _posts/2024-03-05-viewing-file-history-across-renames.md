---
layout: post
title: Viewing file history across renames
date: 2024-03-05 02:12 +0200
categories: blog
---

This entry is also about an addition in Emacs's VC support
(development branch, Emacs 30).

Git doesn't really track renames, but supports printing the commit log
across renames using certain heuristics (seeing that a commit contains
a deleted and a created file where a big enough percentage of lines
are the same; copies are detected similarly), but this has to be
enabled explicitly (the `--follow` option), and it can end up missing
commits in some situations with [complex branch merging
history](https://stackoverflow.com/questions/46487476/git-log-follow-graph-skips-commits).

In Emacs `vc-print-log` (<kbd>C-x v l</kbd>) likewise had problems
with renames, with [multiple](https://debbugs.gnu.org/8756)
[reports](https://debbugs.gnu.org/19045) [over the
years](https://debbugs.gnu.org/55871) and different approaches having been
proposed. Adding the support for `--follow` helped somewhat, but the
commands to view the diff (<kbd>d</kbd>), or annotate/blame
(<kbd>a</kbd>), or check out that old version (<kbd>f</kbd>), stayed
broken when the version had a different file name.

This difficulty is not specific to Emacs, though, and looking around,
Github has recently [adopted a solution for
this](https://github.blog/changelog/2022-06-06-view-commit-history-across-file-renames-and-moves/):
adding a "show history for that name" link when the current log ends.
It's not an original invention either: it started out as a [Chrome
extension](https://github.com/jeffstieler/github-follow-extension).

Anyway, now we also have a button that allows you to jump to the
history of the previous file name. Just make sure to set
`vc-git-print-log-follow` back to its default value (`nil`), to be
able to use it.

![bottom of vc-print-log for vc-git.el](/assets/images/2024-03-05.png)

This is currently implemented for Git and Mercurial only.
