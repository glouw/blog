---
layout: post
---

GNU Makefiles are like Latin. Only scholars can speak Latin.
Thankfully patterns exist in GNU Makefiles, and like Latin, a good template is all one needs:

    BIN = a.out

    SRCS = main.c foo.c bar.c
    OBJS = $(SRCS:.c=.o)
    DEPS = $(SRCS:.c=.d)

    CFLAGS = -Wshadow -Wall -Wpedantic -Wextra -g

    LDFLAGS = -lm

    $(BIN): $(OBJS)
        gcc $(CFLAGS) $(OBJS) $(LDFLAGS) -o $(BIN)

    %.o : %.c Makefile
        gcc $(CFLAGS) -MMD -MP -MT $@ -MF $*.td -c $<
        mv -f $*.td $*.d

    %.d: ;
    -include *.d

    clean:
        rm -f $(BIN) $(OBJS) $(DEPS)


# So whats going on here?

While compiling, file header dependencies are stored in dependency files. If a source file is updated
it is solely recompiled and relinked. If any header is updated, all source files which included the
header are recompiled and relinked. If the Makefile is updated everything is recompiled and relinked.

# So what do I do?

Simply add your new source files to SRCS, libraries to LDFLAGS, and more compiler flags like -O3 and -flto to CFLAGS.
