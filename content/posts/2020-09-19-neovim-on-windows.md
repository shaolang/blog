---
title: "Neovim on Windows"
date: 2020-09-19T18:54:39+08:00
allowComments: true
---

I may be missing something but Neovim requires some additional configuration
to make it work on Windows; the following are the things I've done to
make Neovim work on a Windows machine that also makes it co-exist with
Vim.[^1] After [downloading Neovim][download] and unzipping that to somewhere,
the first thing to do is to configure it through `init.vim`.

## `init.vim` aka Neovim's `.vim`/`_vimrc`
Most online articles say `init.vim` should reside at `~/.config/nvim/init.vim`
and, well, we know `~` translates to `%userprofile%` (or `C:\Users\username`)
in Windows. Nope, in Windows, it should be `~/AppData/Local/nvim/init.vim`,
as an [answer][answer] on Stack Overflow says.

Add the following to `init.vim` to make Neovim reuse Vim's configurations,
thus easing the transition from and making it co-exists with Vim:

```viml {linenos=table}
set runtimepath+=~/vimfiles,~/vimfiles/after
set packpath+=~/vimfiles
source ~/_vimrc
```

The script above assumes Vim runtime files exist at `~/vimfiles`; amend that
accordingly to your setup.

The last line in the script loads Vim's configuration from `~/_vimrc` (so
change that too if yours exists elsewhere).

## Runtime path
And within the Stack Overflow answer (and everywhere else), someone mentioned
that the location is also stated in the help file, "With `:help init.vim`, you
can get the location..." Trying that, Neovim helpfully tells me "Sorry, no
help for init.vim".

Similar to vim, Neovim has its own set of runtime files. Although Neovim
bundles them in its distribution, it puts them at `Neovim/share/nvim/runtime`
and doesn't include it in the default [runtimepath][runtimepath]. Instead of
copying the contents in `runtime` over to `Neovim/bin`, add that to the
`runtimepath` in `init.vim`:

```viml {linenos=table, hl_lines=[1]}
set runtimepath+=~/vimfiles,~/vimfiles/after,c:/neovim/share/nvim/runtime
set packpath+=~/vimfiles
source ~/_vimrc
```

Notice that Neovim's runtime directory is appended to the first line. Again,
change that to point to the directory where you've unzipped Neovim.

And that completes the setup required to run Neovim on Windows.

[^1]: [Chocolatey][chocolatey] and [Scoop][scoop] might have made these an
non-issue, but I didn't try them.

[download]: https://github.com/neovim/neovim/releases/latest
[chocolatey]: https://chocolatey.org
[scoop]: https://sccop.sh
[answer]: https://stackoverflow.com/a/45219657
[runtimepath]: https://medium.com/usevim/vim-101-runtimepath-83194d411b0a
