---
title: "Keeping organized, the Vim way"
layout: post
---

Over the past years the amount of things I need to keep track of has grown a lot. Since I'm well aware of the limited space inside my skull, I've started to write things down. 

## First version: plain markdown files
Since I tend to spend a lot of my time in Vim, my first system to keep on top of things was simple: I started keeping a list of markdown files, with a simple TODO.md split into 2 sections: TODO and DONE.

The TODO section is a simple list of bullets:

```
  TODO
  ====
    * Mail Raymond
    * Write documentation
    * Read up on project X

  DONE
  ====
    * Make car appointment
    * Buy birthday present
```

When I finished a task, I'd simple cut the line with `dd` and paste it again under DONE. Nice and simple. 

This soon evolved into having a `personal.md`, a `JoC.md` (for the Joy of Coding conference), and some work-related files for the clients I work for.

## Looking further

This system has its limits though, and lately I've been reading a lot about [Orgmode](https://orgmode.org/).
If you haven't heard of it, orgmode is a project for Emacs, which does a lot of things: it's a document authoring system, an agenda manager and a lot more.

A really good article about this (and a definite must-read) is [Org Mode - Organize Your Life In Plain Text!]( http://doc.norang.ca/org-mode.html ) by Bernt Hansen. It goes into great depth about his workflow, which is very well thought out but much more than I need.

I'm not quite willing to leave Vim for Emacs yet though (though I might explore SpaceMacs at some point), so I've been looking for Vim alternatives.

Here's what I've found so far:

  * [Vim org-mode](https://github.com/jceb/vim-orgmod) ⇒ A re-implementation of org-mode for Vim. It's not feature complete, but it gets the core right.
  * [Vim Organizer](https://github.com/hsitz/VimOrganizer) ⇒ Instead of trying to re-implement org-mode, Vim Organizer works as a front-end to the actual org-mode. The main downside of this is needing to run Emacs as a service.
  * [Do too](https://github.com/dhruvasagar/vim-dotoo) ⇒ Not an actual org-mode implementation but a work-alike, based on Bernt Hansen's article. The file-format is mostly but not completely compatible. Since it's based on the article it's much more opinionated.
