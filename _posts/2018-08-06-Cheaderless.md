---
layout: post
---

Headerless C99. It can be done given a basic template:

    #ifndef __CLASS__
    #define __CLASS__

    #undef defines
    #ifdef CLASS
        #define defines(...);
    #else
        #define defines(...){__VA_ARGS__}
    #endif

    #endif

Let us model a wristwatch as WATCH.c:

    #ifndef __WATCH__
    #define __WATCH__

    #include <stdio.h>

    #undef defines
    #ifdef WATCH
        #define defines(...);
    #else
        #define defines(...){__VA_ARGS__}
    #endif

    typedef struct
    {
        int hour;
        int minutes;
        int seconds;
    }
    Watch;

    Watch wnew(void) defines
    (
        Watch w;
        w.hour = 4;
        w.minutes = 20;
        w.seconds = 0;
        return w;
    )

    void wtell(const Watch w) defines
    (
        printf("%d\n", w.hour);
        printf("%d\n", w.minutes);
        printf("%d\n", w.seconds);
    )

    #endif

Our wristwatch object can be compiled into an object file with:

    gcc WATCH.c -c

And its inclusion into another file can be done with:

    #define WATCH
    #include "WATCH.c"

    int main()
    {
        Watch watch = wnew();
        wtell(watch);
    }

The lines:

    #define WATCH
    #include "WATCH.c"

serve as an import statement.

Our final binary may be compiled as:

    gcc -c main.c WATCH.o

As the system model grows, a Makefile can track .c file dependency inclusions with the
-MMD -MP -MT -MF flags and recompile changed .c files and dependencies thereof. For such an
advanced build using more than one object for a system, see [Cheaderless](https://github.com/glouw/cheaderless).

The makefile it uses is described here: [Good Makefiles](http://glouw.com/2018/04/14/Good-Makefiles.html).
Touching main.c, for instance (as it is dependency free), will recompile main.c and leave the PERSON and WATCH
.c files alone and relink their object files.

Now, if only there were a way to have the import statement:

    #define WATCH
    #include "WATCH.c"

defined as:

    #import WATCH

then this would be a fun system to maintain!

Alas, there is no way to run the preprocessor twice, and the addition of a secondary custom preprocessor
is fringing upon new language territory.
