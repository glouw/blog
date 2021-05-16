---
layout: post
---

Often times one needs to obfuscate game assets. If a game relies on BMP files, SDL2,
and vim, the first step to obfuscating the assets without the need of cryptography is already done.

SDL2 has out of the box support for Windows BMP files. The macro to load BMP files is declared in
`SDL_Surface.h` and takes a file path string as an argument:

    #define SDL_LoadBMP(file) SDL_LoadBMP_RW(SDL_RWFromFile(file, "rb"), 1)

By dissecting the contents of `SDL_LoadBMP` we learn `SDL_RWFromFile` returns an `SDL_RWops` pointer.
Fortunately, an alternative declaration in `SDL_rwops.h` exists, and it allows for a generic read from raw memory:

    extern DECLSPEC SDL_RWops *SDLCALL SDL_RWFromMem(void *mem, int size);

Combining this call with `SDL_LoadBMP_RW` forms a new definition, `SDL_LoadBMPFromMem`, and it provides
a facility for loading BMP files from memory:

    #define SDL_LoadBMPFromMem(mem, size) SDL_LoadBMP_RW(SDL_RWFromMem(mem, size), 0)

Thankfully, a hex dumper named `xxd` comes pre-installed with vim and it includes a huge time saving flag
named `-include` for transforming a file (in this case, a BMP file) into a C style array with a size constant:

    xxd -include Image.bmp

The output prints directly to stdout, printing both the `unsigned char` array `Image.bmp` and the length
of the array:

    unsigned char Image_bmp[] = {
    ...
        0xeb, 0x01, 0x40, 0x33, 0x33, 0x13, 0x80, 0x66, 0x66, 0x26, 0x40, 0x66,
        0x66, 0x06, 0xa0, 0x99, 0x99, 0x09, 0x3c, 0x0a, 0xd7, 0x03, 0x24, 0x5c,
        0x8f, 0x32, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    ...
    };
    unsigned int Image_bmp_len = 373;

Loading the BMP image is just a matter of using the newly crafted `SDL_LoadBMPFromMem` macro:

    SDL_Surface* arrow = SDL_LoadBMPFromMem(Image_bmp, Image_bmp_len)

With `xxd` able to reverse BMP images, and SDL2 being able to natively load BMP images
without the need of external dependencies like `SDL_image`, this solution is the first step
to obfuscating important art assets in a lightweight fashion that would otherwise spoil a game's plot.
Yes, the game binary will bloat, and the BMP can still be reversed with an `objdump` of the binary,
but by hiding the BMP in the binary the secret asset will be hidden to most of the players.

Of course, if the game becomes really popular, chances are it will be disassembled, and the Streisand Effect
may otherwise further publicize the assets that were once obfuscated!
