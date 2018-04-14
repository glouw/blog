---
layout: post
---

Makefiles are like Latin. Only scholars can speak Latin. Thankfully patterns exist in Makefiles,
an like Latin, a good template is all one needs:

    CC = gcc

    BIN = a.out

    SRCS = main.c foo.c bar.c

    CFLAGS = -Wshadow -Wall -Wpedantic -Wextra -Wdouble-promotion -g

    LDFLAGS = -lm

    $(BIN): $(SRCS:.c=.o)
        $(CC) $(CFLAGS) $(SRCS:.c=.o) $(LDFLAGS) -o $(BIN)

    %.o : %.c Makefile
        $(CC) $(CFLAGS) -MMD -MP -MT $@ -MF $*.td -c $<
        mv -f $*.td $*.d

    %.d: ;
    -include *.d

    clean:
        rm -f $(BIN) $(SRCS:.c=.o) $(SRCS:.c=.d)

# Whats going on here?

While compiling, file header dependencies are stored in dependency files. If any header is updated, all source files which
included the header are recompiled and relinked into the final binary.

# So what do I do?

Simply add your new source files to SRCS and add libraries to LDFLAGS.

# Is it C++ compatible?

Yes, change CC to g++, and replace all occurrences of SRCS:.c=.d with SRCS:.cpp=.d.
