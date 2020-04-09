---
layout: post
---

This series will be dedicated to rewriting Age of Empires II - The Age of Kings.

## Introduction

The goal of this series is to implement an 8 player multiplayer map, starting each player in the dark age as they
work their way into the Castle age. Certain simplifications, like omitting terrain elevations and civilizations,
will be made to reduce overall complexity of the source.

This series will be using your standard ISO C99 compiler, SDL2, and a Makefile.

For legal reasons, the art data files will not be supplied with the source rewrite, but can be attained
by downloading and installing (with Wine if on Linux) the [Age of Empires 2 - The Age of Kings Trial](https://archive.org/download/AgeofEmpiresIITheAgeofKings_1020/AoE2demo.zip).

Without further adieu, let's get started.
