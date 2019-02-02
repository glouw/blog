---
layout: post
---

GNU Makefiles and C. One just needs a good template:

    BIN = a.out

    SRCS = main.c foo.c bar.c
    OBJS = $(SRCS:.c=.o)
    DEPS = $(SRCS:.c=.d)

    CC = gcc

    CFLAGS = -std=c99 -Wshadow -Wall -pedantic -Wextra -g -O3 -flto -march=native

    LDFLAGS = -lm

    #
    # Linker.
    #

    $(BIN): $(OBJS)
    	$(CC) $(CFLAGS) $(OBJS) $(LDFLAGS) -o $(BIN)

    #
    # Compiler.
    #

    %.o : %.c Makefile
    	$(CC) $(CFLAGS) -MMD -MP -MT $@ -MF $*.td -c $<
    	mv -f $*.td $*.d

    #
    # Dependency Generator.
    #

    -include *.d

    .PHONY: clean
    clean:
    	rm -f $(BIN) $(OBJS) $(DEPS)


# So whats going on here?

While compiling, file header dependencies are stored in dependency files. If a source file is updated
it is solely recompiled and relinked. If any header is updated, all source files which included the
header are recompiled and relinked. If the Makefile is updated everything is recompiled and relinked.

# So what do I do?

Simply add your new source files to SRCS, libraries to LDFLAGS, and more compiler flags to CFLAGS.

By the way, check out the [Makefile for Andvaranaut.](https://github.com/glouw/andvaranaut/blob/master/src/Makefile)

It uses this Makefile template, and is able to compile Andvaranaut using either C or C++ compilers
for both gcc and clang.
