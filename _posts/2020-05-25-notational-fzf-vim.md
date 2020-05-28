---
layout: post
category: software
tags: notetaking vim tools markdown fzf ripgrep
date: 2020-05-25 18:11:00
description: >-
  I've tried vimwiki, OneNote, Bear, Gollum, Notion, you name it. Today I'm
  going to share the system I'm using now, and what my requirements were.
title: Fast, Git-backed, Markdown notes in Vim with notational-fzf-vim
---

Like many developers, I like to take notes during the day. And like many
developers, I spent a lot of time trying out different apps and systems for
notetaking, especially during the last couple years. I've tried vimwiki,
OneNote, Bear, Gollum, Notion, you name it. Today I'm going to share the system
I'm using now, and what my requirements were.

<!-- more -->

**TLDR:** [notational-fzf-vim][3]

## Requirements

These are must-haves for me, and are responsible for my jumping from system to
system. It's difficult to find different apps that tick all these boxes:

- Notes must be in Markdown files.
- I must be able to easily back up all notes with Git.
- I must be able to use Vim or Vim-like keybindings.
- Notetaking app must not look like shit.
- Notes cannot be locked up within en vogue SaaS app, even if it has a
  (probably garbage) export option.
- I want something more than just manually managing text files.

It would also be cool to be able to create links between notes, but only if
they auto-updated when I renamed or moved files. But I could look past a
solution not having linking.

During my long search, nothing I tried was quite satisfying. I was extremely
frustrated because it seemed like what I wanted was _pretty simple_ and I felt
like it _should_ exist, and I knew it _could_ exist because [Notational
Velocity][1] and [nvAlt][2] exist, but they didn't satisfy all my requirements.
I had started building my own solution in Ruby when I found
[notational-fzf-vim][3].

## Solution: notational-fzf-vim

[notational-fzf-vim][3] is a simple Vim plugin that gives you Notational
Velocity-like functionality, powered by [fzf][4] and [ripgrep][5], and nothing
more. Hit your keybinding and search for a file in your configured directories,
or create one if it doesn't exist. And it's lightning-fast.

I'm thrilled to have found it. On top of that, I discovered fzf and ripgrep,
two awesome utilities that I had never seen before and now use daily.

If you're interested, there's a good readme in the [git repo][3] that will hook
you up proper. It speaks for itself; don't forget dependencies and required
settings. Follow those instructions, and then you'll probably want to set a
keybinding. Here's mine, which activates notational-fzf-vim when I hit `F3` in
normal, visual, or insert mode (`F3` is fast and easy with my keyboard config):

```vim
noremap <silent> <F3> :NV<CR>
vnoremap <silent> <F3> <C-C>:NV<CR>
inoremap <silent> <F3> <C-O>:NV<CR>
```

notational-fzf-vim allows me to satisfy all my requirements. Plus I can use the
vim environment I already have, and do Git backups however I want. It doesn't
have the optional linking I mentioned, but I'm cool with that because using
links would actually be _slower_ than hitting my `:NV` binding and typing. I
know, because I used vimwiki and this is way faster.

## Feedback

Questions, comments, or tips for me? See a mistake in this post? [Send me an
email.][6]


[1]: http://notational.net/
[2]: https://brettterpstra.com/projects/nvalt/
[3]: https://github.com/alok/notational-fzf-vim
[4]: https://github.com/junegunn/fzf
[5]: https://github.com/BurntSushi/ripgrep
[6]: mailto:hello@davidgay.org
