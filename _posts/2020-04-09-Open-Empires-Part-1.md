---
layout: post
---

The goal of this series is to walk you, the reader, through implementing an 8 player RTS lockstep engine using the assets
from Trial version of Age of Empires 2. Certain simplifications, like omitting terrain elevations and civilizations,
will be made to reduce overall complexity of the source.

This series will be using your standard ISO C99 compiler, SDL2, and a Makefile.

For legal reasons, the art data files will not be supplied with the source rewrite, but can be attained
by downloading and installing (with Wine if on Linux) the [Age of Empires 2 - The Age of Kings Trial](https://archive.org/download/AgeofEmpiresIITheAgeofKings_1020/AoE2demo.zip).

Below is a quick screen grab of the latest development progress (as of April 9, 2020), but without further adieu, let's get started.

![Salesman](/images/openempires/img1.png)
