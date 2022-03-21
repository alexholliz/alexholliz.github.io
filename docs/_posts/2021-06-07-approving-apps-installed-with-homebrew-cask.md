---
layout: post
title:  "Approving Apps Installed with Homebrew Cask"
date:   2021-06-07 17:34:12 -0800
categories: mac homebrew cask snippet
---

I use *[Homebrew](https://brew.sh/)* on my mac to manage packages. Homebrew *[Cask](https://github.com/Homebrew/homebrew-cask)* is a way to extend homebrew to manage packaged binaries. There's an annoying (and correct) problem that when you install something like iterm2-nightly, or chromium, or mpv with homebrew cask, the first time you try to run it, you'll get a warning that the developer cannot be verified, and then you're given the option to delete the app, or close the warning. But the app doesn't start.

There are a few ways to deal with this:

1. Install it without homebrew in the first place!
    ```
    brew install --no-quarantine chromium
    ```

    or

    ```
    brew cask install --no-quarantine chromium
    ```

    <https://github.com/Homebrew/homebrew-cask/blob/HEAD/USAGE.md#options>

    and

    <https://docs.brew.sh/FAQ#why-cant-i-open-a-mac-app-from-an-unidentified-developer>

1. Find the app's install dir, right click on the .app file, and hit "Open"

2. From the terminal, take it out of jail:
    ```
    xattr -d com.apple.quarantine /Applications/Chromium.app
    ```

3. Remove the Quarantine Flag set by homebrew:
    ```
    xattr -cr /Applications/Chromium.app
    ```