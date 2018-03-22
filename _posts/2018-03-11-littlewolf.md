---
layout: post
---

[GitHub Source](https://github.com/glouw/littlewolf)

![](/images/lw/peekgif.gif)

Littlewolf recently gained a bit of popularity on HackerNews so a blog post was due.
Littlewolf is a Wolfenstein a software graphics engine written to be as minimal as possible. It strives to emulate
Wolfenstein and Doom with a technique known as raycasting.

# Ray Casting

From the player's position, a ray is cast within in the player's field of view (fov) for each vertical pixel column of the screen.

![](/images/lw/1.PNG)

The distance between each ray in relation to the field of view is equal. This is accomplished by linear
interpolating the field of view with an x percentage of the x-resolution.

    const Point direction = lerp(camera, x / (float) gpu.xres);

This yields one directional vector for each column the screen. Assuming the screen resolution is 700 pixels wide,
distances d0 - d699 (inclusive) are generated as seen in the image above.

A ray jumps both a vertical square and a horizontal square. The shortest distance of the two is the correct jump of the ray.
Seen below, B is a horizontal jump at (0.8, 1.0) and B1 is a vertical jump at (1.0, 1.23). B is closest and is chosen
for the next jump iteration.

![](/images/lw/2.PNG)

This continues until a wall is hit. Walls are marked by numbers to indicate their color. Seen here is a red wall.
Assume the red wall stretches on indefinitely.

![](/images/lw/3.PNG)

Due the nature of typecasting floating point numbers to integers, a wall may be missed if not probed.
Probes are small vector extensions of the ray, extending either vertically, horizontally, or diagonally
depending on the wall hit location.

If a vertical wall is hit, a horizontal probe (dx) is used to get the wall color index. Likewise, horizontal wall hits use a vertical probe (dy).

![](/images/lw/4.PNG)

This is accomplished by checking the floating point decimal value for zero. Wall hits between two walls uses a diagonal probe (dc), else there will be contention between the color indices.

![](/images/lw/5.PNG)

This is calculated by subtracting the horizontal vector from the vertical vector. If the resulting vector magnitude is small, the ray is diagonal,
ultimately yielding the expression for three probe cases as:

    const Point probe = mag(sub(hor, ver)) < 1e-3f ? dc dec(ray.x) == 0.0f ? dx : dy;

# Wall Height

The height of the wall is calculated by taking the normal ray to the wall (in this case, the x direction), and multiplying the
focal length of the field of view (0.8 from before in the first image). The normal is clamped to a small value in case the player
gets too close to the screen.

    const float normal = ray.x < 1e-2f ? 1e-2f : ray.x;
    const float size = 0.5f * focal * xres / normal;

The ray is defined as the difference of the wall hit and the player:

    const Ray ray = sub(wall.hit.where, hero.where);

This works when the theta of the camera is set to zero. Given the camera is rotated,
the ray index hit is to be sampled and the resulting ray is to be corrected back to the fov line:

    const Point corrected = turn(ray, -hero.theta);
    const float normal = corrected.x < 1e-2f ? 1e-2f : corrected.x;
    const float size = 0.5f * focal * xres / normal;

The top and bottom of the wall can be found by subtracting half the size of the wall from the middle of the screen's y-resolution.

    const int top = (yres + size) / 2.0f;
    const int bot = (yres - size) / 2.0f;

Rasterization is done:

![](/images/lw/9.PNG)

# Ceiling and Floor casting

Ceiling and floor casting require a percentage of the floor in relation to the wall height. By dividing the wall in two an expression for y can be
found for everything below the middle of the wall:

![](/images/lw/12.PNG)

An expression for y is formed:

    y = yres / 2 - h / 2

Where h is the size of the wall calculated earlier. With a bit of reordering a percentage expression for y is found.

    y - yres / 2 = -h / 2

    1.0 = -(h / 2) / (y - yres / 2)

    1.0 = -h / (2 * y - yres)

Yielding:

    const float percentage = -size / (2 * (y + 1) - yres);

Where one was added to y to account for any floating point error for the flooring and ceiling with longer distances.
The above equation can be wrapped into a function called pcast():

    static float pcast(const float size, const int yres, const int y)
    {
        return size / (2 * (y + 1) - yres);
    }

By dropping the negative from the call, the call can be dual purposed for ceiling and floor casting by prefixing the call with a negative.

The trace line between the two points from the hero location (origin here) and the wall hit is declared:

    const Line trace = { hero.where, hit.where };

The trace is lerped() with the percentage from the pcast equation. The tile index of the ceiling and flooring tiles is sampled
and the pixel is put() to the screen.

    for(int y = 0; y < wall.bot; y++)
        put(display, x, y,
            color(tile(
                lerp(trace, -pcast(wall.size, gpu.yres, y)), map.floring)));

The same is done for the ceiling. Notice the positive before the pcast() call.

    for(int y = wall.top; y < gpu.yres; y++)
        put(display, x, y,
            color(tile(
                lerp(trace, +pcast(wall.size, gpu.yres, y)), map.ceiling)));

![](/images/lw/11.PNG)

Not too shabby. [Give the source a read.](https://github.com/glouw/littlewolf)
