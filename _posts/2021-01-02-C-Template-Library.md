---
layout: post
---

[GitHub Source](https://github.com/glouw/ctl)

A recent round of small projects left me rewriting the same set of linked lists and
resizable arrays for a myriad of types. While enjoyable to most C programmers, rewriting the
same data structure is definitely a time sink, and the conventional approach
of [encapsulating void pointers](https://glouw.com/2020/07/13/Generic-Hash-Tables.html)
is slow and prone to type errors.

The gist to effective compile time polymorphism lies within the preprocessor:

    #define T int

    T sum(T* array, size_t size)
    {
        T total = 0;
        for(size_t i = 0; i < size; i++)
            total += array[i];
        return total;
    }

The caveat being, names require a type signature, expanding the above to
`int int_sum(int* array, size_t size)`:

    #define CAT(a, b) a##b
    #define PASTE(a, b) CAT(a, b)
    #define JOIN(prefix, name) PASTE(prefix, PASTE(_, name))

    #define T int

    T JOIN(T, sum)(T* array, size_t size)
    {
        T total = 0;
        for(size_t i = 0; i < size; i++)
            total += array[i];
        return total;
    }

Wrapping the `sum` function as a `static inline` into a single header named
`sum.h` yields an infinite number of expansion possibilities
(granted, `T` is undefined within `sum.h`):

    #define CAT(a, b) a##b
    #define PASTE(a, b) CAT(a, b)
    #define JOIN(prefix, name) PASTE(prefix, PASTE(_, name))

    #define T int
    #include "sum.h"

    #define T float
    #include "sum.h"

    #define T double
    #include "sum.h"

    #define T char
    #include "sum.h"

I pushed this concept to the extreme, and backported the core of the C++11 STL
to ISO C99/C11. As per a measure of safety, each push to `master` or `staging` kicks
off a 3 hour test suite which randomly runs each container function against the STL
and checks for inconsistencies. As of writing, `unordered_set` is not implemented -
the holidays has effectively put me in vacation mode:

[GitHub Source](https://github.com/glouw/ctl)
