# Welcome

Welcome to this tutorial!

It is designed to accompany `aarch-os`, guiding you through the basics of OS development.
This is not meant to be a comprehensive guide, and we will not cover the entirety of the codebase.
Instead, this should serve as a theoretical introduction to the topic 
and a starting point for your own exploration.

Do try to follow along with the code and experiment with it for the best learning experience.

> This tutorial will use `zsh` on macOS.
> 
> Everything should be reproducible in a Linux environment, but the necessary configuration 
> steps and some commands may differ. If you are on Windows, I strongly recommend using WSL.
> 
> I will mostly use Homebrew to install software, but most of it is available from 
> other sources as well.
{style="note"}


## Prerequisites
To get started, ensure you have the basic tools, such as `make` and `git` installed.
These are all nicely packaged in the XCode Command Line Tools on macOS, 
so you can get them simply by running the following command:

```Bash
xcode-select --install
```
{prompt="$"}

While these do come with a version of GCC, we will need a different one
to cross-compile for a non-standard target, bare metal aarch64 (`aarch64-none-elf`).

This toolchain is provided by ARM as part of the
[Arm GNU Toolchain](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain),
and is available from Homebrew as a Cask:

```Bash
brew install --cask gcc-aarch64-embedded
```
{prompt="$"}