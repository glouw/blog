---
layout: post
---

[GitHub Source](https://github.com/glouw/softshader)

I ported a couple popular Shadertoy shaders to run on the CPU:

Sea Scape, by Alex Alekseev (shadertoy.com/user/TDM)

![seascape](/images/ss/seascape.png)

Creation, by Danilo Guanabara (shadertoy.com/user/Danguafer)

![creation](/images/ss/creation.png)

Tunnel, by Inigo Quilez (shadertoy.com/user/iq)

![tunnel](/images/ss/tunnel.png)

The math escapes me, but I used this exercise as an introduction to C++11.
The shaders request a single uint32_t pointer from SDL2 and then proceeds to divide
the screen into as many horizontal rows as there are logical cores on your CPU. Each
render row is rendered in tandem with the others.

The performance is strong for the creation and tunnel shaders, averaging
60 FPS with vertical sync, but the seascape shader runs at a steady 0.5 FPS, even without
antialiasing.

A cool exercise nonetheless. I have never seen water rendered on the CPU looking this good,
but at this point, even on an i5-3320M with 4 logical cores, this renderer may as well be a
ray tracer with its 0.5 FPS render speed.

Anyway, checkout the source if you like. The creation and tunnel shaders are totally worth
studying. I will maybe return one day and breakdown the math behind the seascape shader.
