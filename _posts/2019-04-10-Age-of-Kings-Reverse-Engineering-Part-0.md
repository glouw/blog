---
layout: post
---

This series will be dedicated to reverse engineering Age of Empires II - The Age of Kings.

    Part 0: Introduction and tooling
    Part 1: Data loading and unpacking
    Part 2: Terrain placement and tilemap editing
    Part X: ... (TBD)

## Introduction

The goal of this series is to implement an 8 player multiplayer map, starting each player in the dark age as they
work their way into the castle age with either the Briton or Celt factions. Certain simplifications,
like omitting terrain elevations, will be made to reduce overall source complexity.

As for the reverse engineering, Ensemble opted to store all artistic assets into a binary DRS database.
A DRS database contains sprite bitmaps, sound wav files, and static GUI bitmaps.

The engine loads the binary DRS database into RAM, and at runtime utilizes the CPU to swap out sprite palette colors.
This gives the effect of 8 player color for the sprites, sprite outlines of differing color when unit a sprite is behind
a building sprite, and checkered effects when building sprites are placed on water.

Unit movement and combat is grid based with a modified version of the a-star pathfinder, while as the renderer
is simply a painter's algorithm that first projects sprites with the lowest y-coordinate onto an isometric tile grid.

A single music track loops during gameplay, and small sound wav snippets are played when villagers, buildings, units,
and boats are selected.

## Tooling

This series will be using your standard ISO C99 compiler, SDL2, and a Makefile.

Source code will not ship with the game binary DRS database.
You will need to install the Age of Empires 2 - The Age of Kings Demo. The installation also works in wine if you are in Linux.

Reverse engineering documentation will be referenced from OpenAge's efforts to fully reverse engineer the Age of Empires II and all its expansions.
I suggest you check them out.

Without further adieu, let's get started.
