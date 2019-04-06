---
layout: post
---

From `terrain.drs` exists a series of terrain files. `File 0` may be dirt, `File 1` grass, and so on.

Each `File` has roughly a hundred animation frames that, when unpacked and lined up, create
one large tile roughly ten times the area.

Programmatically, map data is stored in a 2D array of integers. At runtime, this 2D
array is projected isometrically onto the screen like:

                       +
    +--------+        /d\
    |a b c d |       /c h\
    |e f g h | -->  /b g l\
    |i j k l | --> +a f k p+
    |m n o p |      \e j o/
    +--------+       \i n/
                      \m/
                       +

Given array values `a -> p` are all `integers`, and equal to `1`, they will all render
a mega grass tile.

`https://github.com/glouw/aoklite/blob/master/src/Sdl.c`

The animation to use, however, is dictated by:

    int bound = sqrt(tile_count) // Roughly 100 for tile_count
    int animation = (x % bound) + ((y % bound) * bound);

Given the width and height of each tile is known, and a projection
transformation of:

    int xx = (y + x) * scale.width / 2;
    int yy = (y - x) * scale.height / 2;

Where `x` and `y` are map coordinates, and `xx` and `yy` projected isometric coordinates,
the final render with a mix of grass and dirt tiles may look something like:

![](/images/aoklite/2019-04-06-053120_1000x600_scrot.png)

Of course, this does not include tile blending.

Likewise, if a point is to be clicked on screen, the reverse projection calculations are
to be applied to query the map.
