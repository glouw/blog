---
layout: post
---

This multi-part series is a walk-through on implementing an 8 player Real Time Strategy (RTS) lockstep engine using the assets
from Trial version of Age of Empires II. Certain simplifications, like omitting terrain elevations and civilizations,
will be made to reduce overall complexity of the source.

This tool-set in this series will be using a standard ISO C99 compiler, a Makefile, and your standard address and thread sanitizers.
The entirety of the engine, including the networking control system, multithreaded renderer, sound and input system, asset database,
and the pathfinder, will be built from the ground up. Universe and apple pies apart, SDL2 and friends (SDL2_net, SDL2_mixer) will
provide the foundation for cross platform play.

For legal reasons, the art data files will not be supplied with the source rewrite, but can be attained
by downloading and installing (with Wine if on Linux) the [Age of Empires 2 - The Age of Kings Trial](https://archive.org/download/AgeofEmpiresIITheAgeofKings_1020/AoE2demo.zip).

Below is a quick screen grab of the latest development progress (as of April 9, 2020).

![Salesman](/images/openempires/img1.png)

Without further adieu, let's get started.
