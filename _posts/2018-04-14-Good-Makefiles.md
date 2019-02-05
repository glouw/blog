---
layout: post
---

GNU Makefiles and C. One just needs a good template:

    BIN = a.out

    SRCS = main.c foo.c bar.c

    OBJS = $(SRCS:.c=.o)
    DEPS = $(SRCS:.c=.d)

    CC = gcc

    CFLAGS = -Wshadow -Wall -pedantic -Wextra -g -O3 -flto -march=native

    LDLIBS = -lm

    LDFLAGS =

    $(BIN): $(OBJS) $(DEPS)
    	$(CC) $(OBJS) $(LDFLAGS) $(LDLIBS) -o $(BIN)

    %.o : %.c %.d Makefile
    	$(CC) $(CFLAGS) -MMD -MP -MT $@ -MF $*.d -c $<

    -include *.d

    %.d : ;

    .PHONY: clean
    clean:
    	rm -f $(BIN) $(OBJS) $(DEPS)


# So whats going on here?

Dependency files are generated during compilation. If a source file is updated it is solely recompiled and linked.
If any header is updated then the dependency file is referenced and all source files which included the header are recompiled
and then linked. If a dependency file is accidentally deleted then it is regenerated. If the Makefile is updated then all source
files are recompiled and linked.

# So what do I do?

Simply add your new source files to SRCS, compiler flags to CFLAGS, libraries to LDLIBS, and linker flags to LDFLAGS.

# Can I use this with C++?

Yes, change CC to CXX, gcc to g++, CFLAGS to CXXFLAGS, and all .c files to .cpp files.

# How about a real implementation?

Check out the [Makefile for Andvaranaut.](https://github.com/glouw/andvaranaut/blob/master/src/Makefile)

It uses this Makefile template, but written in the intersection of c99 and c++98, making it compilable with
C and C++ compilers using both gcc and clang (and as such, CC and CXX are removed en lieu of the COMPILER variable).
