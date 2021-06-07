---
layout: post
title:  "Approving Apps Installed with Homebrew Cask"
date:   2021-06-07 17:34:12 -0800
categories: mac homebrew cask snippet
---

I use *[Homebrew](https://brew.sh/)* on my mac to manage packages. Homebrew *[Cask](https://github.com/Homebrew/homebrew-cask)* is a way to extend homebrew to manage packaged binaries. There's an annoying (and correct) problem that when you install something like iterm2-nightly, or chromium, or mpv with homebrew cask, the first time you try to run it, you'll get a warning that the developer cannot be verified, and then you're given the option to delete the app, or close the warning. But the app doesn't start.

There are 2 ways to deal with this:

Find the app's install dir, right click on the .app file, and hit "Open"

or


From the terminal, take it out of jail:
```
xattr -d com.apple.quarantine /Applications/Chromium.app
```
