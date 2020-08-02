---
layout: post
---

[Github Source](https://github.com/glouw/paperview)

I recently stumbled upon a small X11 bash script that calls `feh --bg-fill` in a loop over an array of
PNG files to animate the desktop background.
The performance was fairly poor. Cumulative CPU usage was close to 40% on my X230.

As an alternative, I discovered SDL2 provides a generic interface to create a window from a pixel array,
including the base root X11 window used by desktop wallpaper setters.
By opening an X11 display and creating an X11 window, the window can be coerced into a void
pointer an passed to SDL2:

    Display* x11d = XOpenDisplay(NULL);
    Window x11w = RootWindow(x11d, DefaultScreen(x11d));
    SDL_Window* window = SDL_CreateWindowFrom((void*) x11w);

Renders to this window - the desktop wallpaper - can interface directly with GPU memory via an SDL2 renderer:

    SDL_Renderer* renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC);

SDL2 provides bitmap loading functionality to store bitmap images in RAM:

    SDL_Surface* surface = SDL_LoadBMP("frame.bmp");

Moving the bitmap images from RAM to the GPU is done with a single call:

    SDL_Texture* texture = SDL_CreateTextureFromSurface(renderer, surface)

And copying the bitmap from GPU memory to the X11 window - the desktop wallpaper - is done with two SDL2 calls:

    SDL_RenderCopy(renderer, texture, NULL, NULL);
    SDL_RenderPresent(renderer);

The two NULL arguments specifies the render to stretch fill the background wallpaper.

A picture does not do an animated wallpaper any justice, so check it out on youtube:

https://www.youtube.com/watch?v=6ZTiA885bWM

[Also, try giving the source a read](https://github.com/glouw/paperview). A single file (main.c) in less than 150 lines
yields a high performance animated desktop wallpaper at 60 frames per second.

I decided on the name paperview which stems from pre Netflix video services dubbed Pay-Per-View,
and the _viewing_ of _wallpapers_ in an animated fashion.
