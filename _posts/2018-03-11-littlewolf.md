---
layout: post
---

[GitHub Source](https://github.com/glouw/littlewolf)

[Download Littlewolf-1.0.zip (Windows 32/64 bit)](https://github.com/glouw/littlewolf/releases/download/littlewolf-1.0/littlewolf-1.0.zip)

Littlewolf recently gained a bit of popularity on hacker news so a blog post is due.
Littlewolf is a Wolfenstein a software graphics engine written to be as minimal as possible. It strives to emulate
wolfenstein and doom with a technique known as raycasting.

From the player's position, a ray is cast within in the player's field of view (fov) for each vertical pixel column of the screen.

![](/images/lw/1.PNG)

The distance between each ray in relation to the field of view must be equal. This is easily accomplished by linear
interpolating the field of view with an x percentage of the x-resolution.

    const Point direction = lerp(camera, x / (float) gpu.xres);

This yeilds one directional vector for each column the screen. Assuming the screen resolution is 700 pixels wide,
distances d0 - d699 (inclusive) in the image above.

A ray jumps both a vertical square and a horizontal square. The shortest distance of the two is the correct jump of the ray.
Seen below, B is a horizontal jump at (0.8, 1.0) and B1 is a vertical jump at (1.0, 1.23). B is closest and is chosen
for the next jump iteration.

![](/images/lw/2.PNG)

This continues until a wall is hit. Walls are marked by numbers to indicate their color. Seen here is a red wall.
Assume the red wall stretches on indefinately.

![](/images/lw/3.PNG)

Due the nature of casting floating point numbers to integers, a wall may be missed if not probed.
Probes are small vector extensions of the ray, extending either vertically, horizontally, or diagonally
depending on the wall hit location.

If a vertical wall is hit, a horizontal probe is used to get the wall color index. Likewise, horizontal wall hits use a vertical probe.

![](/images/lw/4.PNG)

This is accomplished by checking the floating point decimal value for zero.

    dec(ray.x) == 0.0f ? dx : dy

Wall hits between two walls uses a diagonal probe, else there will be contention between the color indices.

![](/images/lw/5.PNG)

This is calculated by subtracting the horizontal vector from the vertical vector. If the resulting vector magnitude is small, the ray is diagonal:

    mag(sub(hor, ver)) < 1e-3f ? dc

The height of the wall is calculated by taking the normal ray to the wall (in this case, the x direction), and multiplying the
focal length of the field of view (0.8 from before in the first image). The normal is clamped to a small value incase the player
gets too close to the screen else the size will grow infinitly with close to zero division.

    const float normal = ray.x < 1e-2f ? 1e-2f : ray.x;
    const float size = 0.5f * focal * xres / normal;

The top and bottom of the wall can be found by subtracting half the size of the wall from the middle of the screen's y-resolution.

    const int top = (yres + size) / 2.0f;
    const int bot = (yres - size) / 2.0f;

Knowing the wall height for each vertical column of the screen, rasterization may now be done. The resulting image is a little boring, but ceiling
and floor casting will sprucen things up:

![](/images/lw/9.PNG)

Ceiling and floor casting require a percentage of the floor in relation to the wall height. Dividing the wall in two, an expression for y can be
found for everything below the middle of the wall:

![](/images/lw/12.PNG)

Given that:

    y = yres / 2 - h / 2

Where h here is the size of the wall calculated earlier, a percentage for y can be found with a little reordering.

    y - yres / 2 = -h / 2

    1.0 = -(h / 2) / (y - yres / 2)

With a little cleaning:

    1.0 = -h / (2 * y - yres)

This yields:

    const float percentage = -size / (2 * (y + 1) - yres);

Where 1 was added to y account for any floating point error for the flooring and ceiling you'd see with longer distances.
Size is merely h here.

Multiplying this percentage by -1 will yield the equation for ceiling casting.

![](/images/lw/11.PNG)

