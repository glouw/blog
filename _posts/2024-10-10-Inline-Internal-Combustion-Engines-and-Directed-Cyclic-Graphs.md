---
layout: post
---

<iframe width="720" height="405" src="https://www.youtube.com/embed/4Kh9dTn4-vQ" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

(I am not on a plane - I just happen to listen to jet white noise while I work).

Directed Cyclic Graphs (DCG), when traversed breadth first, happen to model internal combustion fluid sim order. Seen above
is an inline 9 engine with plenum intake left, and four exhaust system, ejecting to the atmosphere, before having a turbo
charger make use of the latent heat and pressure at the joint collector.

While this front end is not connected to the audio generator (just yet) in this post I want to outline that a DCG can,
in breadth first order, operate node-to-node in a perfectly parallel order, allowing no piston manifold channel to execute
its fluid or thermodynamic modelling in an out of order setting.

With a widget being a base class, pay careful attention to the polymorphic ability of the DCG - each node connects by with
user input, where shared or weak pointers are automatically deduced, and each node can at runtime be swapped for static volume
(plenum, runner, exhaust, etc), and/or dynamic volume (a piston). The conventional flow math, directed by isentropic choked
flow, moves mass from widget to widget.
