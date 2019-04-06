---
layout: post
---

# Data

Data, be it sprite bitmap animations, wav files, or static GUI bitmaps, are stored in binary DRS databases. Once
the Age of Kings Trial has been installed (under wine, for instance), these DRS databases can be found at:

    /home/gl/.wine/drive_c/Program Files (x86)/Microsoft Games/Age of Empires II Trial/Data/

To name, the three most important DRS files are:

    graphics.drs
    interfac.drs
    terrain.drs

Of these, `graphics.drs` contains all unit and building sprites, `interfac.drs` contains static GUI bitmaps and color palettes,
and `terrain.drs` contains all isometric terrain tiles.

## Loading a DRS Database.

`https://github.com/glouw/aoklite/blob/master/src/Drs.c`

A DRS database is essentially modelled as

    typedef struct
    {
        FILE* fp;
        const char* path;
        char copyright[80];
        char version[8];
        char ftype[24];
        int32_t table_count;
        int32_t file_offset;
        Table* table;
    }
    Drs;

Extra space is reserved for the strings, but their true on-disk lengths are half of that allocated in the struct,
populating the struct like:

    fread(drs.copyright, sizeof(*drs.copyright), 40, drs.fp);
    fread(drs.version, sizeof(*drs.version), 4, drs.fp);
    fread(drs.ftype, sizeof(*drs.ftype), 12, drs.fp);
    fread(&drs.table_count, sizeof(drs.table_count), 1, drs.fp);
    fread(&drs.file_offset, sizeof(drs.file_offset), 1, drs.fp);

A DRS database contains tables. Allocating space for the table is determined by the `table_count` read from the disk:

    drs.table = (Table*) calloc(drs.table_count, sizeof(*drs.table));

Tables immediately follow on-disk, and are modelled as:

    typedef struct
    {
        char file_extension[8];
        int32_t file_info_offset;
        int32_t num_files;
        File* file;
    }
    Table;

Once again, the true size of the string is twice the size of the size on disk to allow for null byte padding. Struct contents
are similarly populated by contiguous calls to fread.

A table contains a number of files, and their structs, once space is allocated, models as:

    typedef struct
    {
        int32_t id;
        int32_t data_offset;
        int32_t size;
    }
    File

The `data_offset`, in bytes, determines where within the DRS database a file format of SLP, WAV, or BINARY is placed.

SLP files contain visual animations. WAV files contains sounds and music. BINARY files contains scripts, or color palettes.

Both terrain and unit sprite / building animations are stored as SLP files within terrain.drs and graphics.drs, respectively.

Static GUI bitmaps are also stored in SLP files, but within interfac.drs.

Nevertheless, given both a table index and file index, either an SLP, WAV, or BINARY file may be loaded from disk.

## Loading an SLP File.

`https://github.com/glouw/aoklite/blob/master/src/Slp.c`

SLP files are modelled as:

    typedef struct
    {
        char version[8];
        int32_t num_frames;
        char comment[48];
        Frame* frame;
        Image* image;
    }
    Slp;

Immediately after reading the header, frames are packed contiguously, and then images.

Frames are modelled as:

    typedef struct
    {
        uint32_t cmd_table_offset;
        uint32_t outline_table_offset;
        uint32_t palette_offset;
        uint32_t properties;
        int32_t width;
        int32_t height;
        int32_t hotspot_x;
        int32_t hotspot_y;
    }
    Frame;

And images are modelled as:

    typedef struct
    {
        Outline* outline_table;
        uint32_t* cmd_table;
        uint8_t* data;
        int height;
        int size;
    }
    Image;


An outline is modelled as:

    typedef struct
    {
        uint16_t left_padding;
        uint16_t right_padding;
    }
    Outline;

Outlines determine the appropriate number of blank space to the left and right of a sprite's animation frame.

The `outline_table_offset` from the `Frame` is an array as large as the frame `height` value.

Using a y-index to index the `outline_table_offset` array will yield an address for indexing
the `data` array of `Image`. This data array will contain an array of color palette indices
and command values dictating the blitting action of the color palette values. The array is finally
terminated with `0xFF`.

`https://github.com/glouw/aoklite/blob/master/src/Scanline.c`

This requires a tiny virtual machine to unpack the image data.

    uint32_tindex = image.cmd_table[y];
    for(;;)
    {
        const uint8_t command = image.data[index++]
        const uint8_t lower_nibble = command & 0x0F;
        const uint8_t upper_nibble = command & 0xF0;

        if(lower_nibble == 0x0F)
            break;

        switch(lower_nibble)
        {
            case 0x0: case 0x4: case 0x8: case 0xC: ... Lesser Block Copy.

            case 0x1: case 0x5: case 0x9: case 0xD: ... Lesser Skip.

            ...

            case 0xA: ... Fill Player Color.

            case 0xB: ... Draw Shadow.

            case 0xE: ... Extended Commands.
                {
                    switch(command)
                    {
                        case 0x4E: ... Outline Player Color.
                    }
                }
        }
    }

Once the color palette values are unpacked, the `palette_offset` from Frame dictates which
palette to load from interfac.drs, BINARY table 0, file 45 + `palette_offset`.
The color palette is standard RGB, and is packed as PAL-JASC:

    typedef struct
    {
        char label[16];
        char version[8];
        uint32_t* color;
        int count;
    }
    Palette;

`https://github.com/glouw/aoklite/blob/master/src/Palette.c`

Unpacking is a string of `fscanf` calls:

    fscanf(fp, "%15s\n", palette.label);
    fscanf(fp, "%7s\n", palette.version);
    fscanf(fp, "%d\n", &palette.count);
    palette.color = (uint32_t*) calloc(palette.count, sizeof(*palette.color));
    for(int i = 0; i < palette.count; i++)
    {
        int r = 0;
        int g = 0;
        int b = 0;
        fscanf(interfac.fp, "%d %d %d\n", &r, &g, &b);
        palette.color[i] = (r << 16) | (g << 8) | b;
    }

With a bit of SDL2 fundamentals, the RGB values are transferred to a surface, and then to a texture.

![](/images/aoklite/Peek-2019-04-06-06-59.gif)
