---
layout: post
---

[GitHub Source](https://github.com/glouw/littlewolf)

[Download Littlewolf-1.0.zip (Windows 32/64 bit)](https://github.com/glouw/littlewolf/releases/download/littlewolf-1.0/littlewolf-1.0.zip)

Littlewolf recently gained a bit of popularity on hacker news, so I figured a post on its inner workings ought to be done.
Littlewolf is a Wolfenstein a software graphics engine written to be as minimal as possible. It strives to emulate
wolfenstein and doom with a technique known as raycasting.

Raycasting is relatively straightforward. From the player's position, a ray is cast within in the player's field of
view (fov) for each vertical pixel column of the screen.

![](/images/lw/1.PNG)

The distance between each ray in relation to the field of view must be equal. This is easily accomplished by linear
interpolating the field of view with an x percentage of the x-resolution.

    const Point direction = lerp(camera, x / (float) gpu.xres);

This yeilds one directional vector for each column the screen. Assuming the screen resolution is 700 pixels wide,
distances d0 - d699 (inclusive) in the image above.

![](/images/lw/2.PNG)

![](/images/lw/3.PNG)

![](/images/lw/4.PNG)

![](/images/lw/5.PNG)

![](/images/lw/9.PNG)

![](/images/lw/12.PNG)

![](/images/lw/11.PNG)

