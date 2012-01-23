---
layout: post
title:  (2012-1-22)
---

Anybody using git to manage text document revisions might benefit from this. I keep all of my research papers in git. I also collaborate on paper writing fairly often, and I may want to see the difference between two revisions. 

My first solution looked like this:

`git diff --word-diff commit1 commit2`

This works, but long lines—which are very common in text documents—run off the edge of the terminal window. On Linux and Mac, the default pager for viewing diffs is _less_, and its stock viewing mode leaves something to be desired. I would like the long lines to be wrapped.

My next attempt was as follows:

`git diff --word-diff commit1 commit2 | fmt`

This version solved the line-wrapping problem, but the result was ugly and difficult to scan. git and fmt don't play well together. If possible, I would like the changes to be color-coded rather than delineated by some ugly text scheme. Although git's support for color highlighting works better for code than long-form text, it's usually good enough that you can work out what changed.

Version 3:

`git config core.pager 'less -+$LESS -FRX'`
git diff --word-diff=color commit1 commit2

Bingo: lines wrap properly, changes are color-coded, and no junk text is introduced by incompatible tools. If you want to use the same pager settings across all of your git repositories, add the "--global" flag to the first command.