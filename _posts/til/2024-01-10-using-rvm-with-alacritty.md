---
title: Using RVM with Alacritty
category: til
layout: post
---

I have have been using Alacritty as my terminal emulator for the past few months, and I am absolutely loving it so far. However, I have recently run into an issue while trying to use RVM (Ruby Version Manager).

When running the command:

```
rvm use <$RUBY_VERSION>
```

I get the following error message:

```
RVM is not a function, selecting rubies with 'rvm use ...' will not work.

You need to change your terminal emulator preferences to allow login shell.
Sometimes it is required to use `/bin/bash --login` as the command.
Please visit https://rvm.io/integration/gnome-terminal/ for an example.
```

So, we need to be in a login shell. Unfortunately, Alacritty does not have a settings GUI [like gnome-terminal](https://rvm.io/integration/gnome-terminal) and similar, and after thorough investigation I couldn't find a way to set this up through Alacritty. We need a different solution.

Luckily, this is possible through the shell itself. You can open a login shell by running the following command (press Ctrl+D to return to the interactive shell):
```
bash --login
```
This also works with `zsh`, or Z-shell:
```
zsh --login
```
Unfortunately for all you fishers out there, I couldn't find a way to to this with `fish`. If you know how, [send me an email!](mailto:andrew@andrewconl.in)

There are some drawbacks to this method.
1. You will have to do this every time you want to use RVM.
2. Commands such as `rvm use` only persist in the login shell, so you will have to stay in there if you want to use RVM, and the version of Ruby that you have selected. I toyed about with using `zsh --login -c "rvm use <$RUBY_VERSION>"`, but this will immediately return you to the interactive shell without any of the changes that you made.
