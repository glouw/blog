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

    clean:
        rm -f $(BIN) $(OBJS) $(DEPS)


# So whats going on here?

While compiling, file header dependencies are stored in dependency files. If a source file is updated
it is solely recompiled and relinked. If any header is updated, all source files which included the
header are recompiled and relinked. If the Makefile is updated everything is recompiled and relinked.

# So what do I do?

Simply add your new source files to SRCS, libraries to LDFLAGS, and more compiler flags to CFLAGS.

# FAQ

Why are dependencies first created at .td and then moved to .d?

    -> In the off chance the build is stopped mid depdedency generation,
       the .td file will be scrapped and the .d file regenerated.

Why are CFLAGS passed to the linker?

    -> The linker requires the link time flag and optimizer flag.
       Instead of creating an extra variable, CFLAGS is reused.

Can I use this with C++?

    -> Yes, change CC = gcc to CPP = g++, CFLAGS to CPPFLAGS, and -std = c99 to -std = c++14.

The Makefile is broken (eg. tabs are spaces).

    -> My editor retabs tabs to 4 spaces.
       I could not figure out how to mix tabs and spaces for this blog post.

Can you go into further detail on the gcc dependency flags (-MMD, -MP, and friends)?

    -> This post goes into pretty good detail on it all:
       http://make.mad-scientist.net/papers/advanced-auto-dependency-generation/#tldr
