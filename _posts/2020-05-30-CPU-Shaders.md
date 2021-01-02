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

The shaders request a single `uint32_t` pointer from SDL2 and then proceeds to divide
the screen into as many horizontal rows as there are logical cores on your CPU. Each
render row is rendered in tandem with the others.

The performance is strong for the creation and tunnel shaders, averaging
60 FPS with vertical sync, but the seascape shader runs at a steady 0.5 FPS, even without
antialiasing. [The full source is here](https://github.com/glouw/softshader).
The creation and tunnel shaders are worth the study:

    #include "softshader.hh"

    static uint32_t shade(const ss::V2 coord)
    {
        const auto per = coord / ss::res;
        auto c = ss::V3 {};
        auto l = 0.f;
        auto z = ss::uptime();
        for(int i = 0; i < 3; i++)
        {
            const auto p = (per - 0.5f) * ss::V2 { ss::res.x / ss::res.y, 1.f };
            l = ss::length(p);
            z += 0.07f;
            const auto uv = per + p / l * (ss::sin(z) + 1.f) * ss::abs(ss::sin(l * 9.f - z * 2.f));
            const auto cc = ss::length(ss::abs(ss::mod(uv, 1.f) - 0.5f));
            c[i] = (cc == 0.f) ? 1.f : (0.01f / cc);
        }
        const auto v = c / l;
        return v.color(ss::uptime());
    }

    int main()
    {
        ss::run(shade);
    }
