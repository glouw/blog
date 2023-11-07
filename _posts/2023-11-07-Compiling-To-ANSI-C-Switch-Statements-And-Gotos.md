---
layout: post
---

[GitHub Source](https://github.com/glouw/switch)

Let us use ANSI-C (C90) to transpile a target language to an intermediate ANSI-C file.

# The Target Language

Our target language, dubbed `switch`, will be reminiscent of the B Programming Language, supporting
a single type `int`, pointers thereto, operators doubly thereof, basic control flow and looping,
and recursion, but of course, barring function pointers - more on this later. `switch` files use the
`.sw` file type, require an entry point `main`, and cannot import other `switch` files.

For the purpose of this post, a `switch` program can wholistically be exemplified with:

`main.sw`
```
int fun()
{
    ret 3;
}

int main()
{
    int x = 1 + fun();
    while(x != 1)
    {
        x = x - 1;
    }
    if(x == 1)
    {
        ret 0;
    }
    ret 1;
}
```
This example provides basic operators, a function call with a return, variables,
loops, and conditional branching. Sparing the complexities of recursive descent parsing,
lexing, and BNF, this post aims to share the workings of `switch`'s transpiled if-statements,
while loops, and function calls / returns.

# The If Statement

Our standard if-statement:

`if.sw`
```
int main()
{
    if(1)
    {
        2;
    }
}

```
This can be realized in transpiled ANSI-C as:

`if.c`
```
#define INT(var) vs[sp++] = var
#define BRZ(label) if(vs[--sp] == 0) goto label
#define BRA(label) goto label
#define POP() --sp
int main(void)
{
    int vs[128] = { 0 };
    register int sp = 0;
    goto main;
main:
	INT(1);
	BRZ(L0);
	INT(2);
	POP();
	BRA(L1);
L0:
L1:
    return 0;
}
```
Our transpiled ANSI-C enters `main` and sets up a variable stack `vs` and
a stack pointer `sp` to track variable pushes and pops to the variable stack.
The `register` keyword prepended to `sp` serves no purpose but to document the
modeling of the stack machine. A `goto main` statement acts as a reset vector, placing
execution at the goto label `main`. An integer of 1 is loaded, and branches to `L0`
should the integer be zero, otherwise, the next instruction executes, eventually branching to `L1`, which
signifies the end of the program. Integer 1 is popped by the `BRZ` instruction.

Within the body of the if-statement lies an expression without assignment, which loads integer 2,
but subsequently pops the integer from the stack. It serves only to have _some form of
expression_ within the block of the if-statement to pretty print the transpile.
Empty blocks are permitted.

# The While Loop

`while.c`
```
int main()
{
    while(1)
    {
        2;
    }
}
```
Our while loop uses the same inner workings as the if statement.
Integer 1 is pushed to the stack, and branches to `L1` (the end
of the program) if it is zero. The loop pushes and pops integer 2
for the exemplary, and a branch to `L0` repeats the loop. In all instances,
integer 1 is popped form the stack with the `BRZ` instruction.

`while.c`
```
#define INT(var) vs[sp++] = var
#define BRZ(label) if(vs[--sp] == 0) goto label
#define BRA(label) goto label
#define POP() --sp
int main(void)
{
    int vs[128] = { 0 };
    register int sp = 0;
    goto main;
main:
L0:
    INT(1);
    BRZ(L1);
    INT(2);
    POP();
    BRA(L0);
L1:
    return 0;
}
```

# The Function Call

`func.sw`
```
int func()
{
    ret 9;
}

int main()
{
    func();
}
```

The function call introduces stacks `fs` and `bs` which track the function
and base pointer stacks, respectively. Additionally, registers `fp`, `fa`, and `rr`,
are introduced, where `fp` tracks pushes and pops to the function and base pointer
stacks, `fa` that stores the pop of `fs` to temporarily obtain a return function address,
and `rr` to temporarily store the return value of a function. `switch` functions do
not support void returns. New instructions, separate to our `if` and `while` examples, are `SET`,
`CAL`, and `RET`. A keen eye notices the intermingling of a switch statement with gotos within `func.c`.

`SET` pushes the current stack frame's stack pointer value `sp` to `bs` at `fp`.
A base pointer allows called functions to accurately reference local variables according
to their zero offset at compile time. This post will not cover the parsing techniques
for managing local variables, their stack pointer values, lvalues, and rvalues,
so `bs` can effectively be ignored.

`CAL` stores the return address in `fs[fp]`, which is the current source line number,
and goes to the call goto label after incrementing `fp`. The return address is a case statement
with the same line number.

`RET` stores the top of the variable stack in the return register `rr`, decrements the
function pointer `fp`, reloads the previous `sp` and `fa` register values from their respective
stacks, and places the return register `rr` on top of the variable stack `vs`. The return function
address, now residing in `fa`, is jumped to by the switch statement after jumping to the `begin` label.

As the program concludes, a `POP()` instruction after the `CAL()` instruction ensures the stack is cleaned up
of integer 9.

`func.c`
```
#define POP() --sp
#define INT(var) vs[sp++] = var

#define SET()          \
    bs[fp] = sp

#define CAL(name)      \
    fs[fp] = __LINE__; \
    fp++;              \
    goto name;         \
    case __LINE__:

#define RET()          \
    rr = vs[sp - 1];   \
    --fp;              \
    sp = bs[fp];       \
    fa = fs[fp];       \
    vs[sp] = rr;       \
    sp++;              \
    goto begin

int main(void) {
    int fs[32] = { 0 };
    int bs[32] = { 0 };
    int vs[128] = { 0 };
    register int fp = 1;
    register int fa = 0;
    register int sp = 0;
    register int rr = 0;
    goto main;
begin:
    switch(fa)
    {
func:
    INT(9);
    RET();
main:
    SET();
    CAL(func);
    POP();
    }
    return 0;
}
```

# Outro
Combine if-statements, while loops, and function calls, and we have a working language (barring
pointers, operators, and everything else that makes a language expressive, of course). Furthermore,
ANSI-C proves to serve as a highly portable stack machine (with a bit of creativity),
which makes a perfect target for compilation (or transpilation if we are being pedantic).
I have a fully functional `switch` compiler/transpiler that you can find on my
[GitHub page](https://github.com/glouw/switch). There are more `switch` examples there to be found,
including string usage, and integer printing (I regulated myself to _only_ having access to libc's
`putchar`, which is accessed through the `$` operator).

And alas, `switch` does not support function pointers without a GNU extension that allows the address of labels
to be taken as values, in example:

```
label:
    0;
    ...
    void* ptr = &&label;
```

Other than that, I leave you with `qsort` transpilable with `switch`, and then it's post
processed intermediate ANSI C file:

`qsort.sw`
```
int swap(int* a, int x, int y)
{
    int temp = *(a + x);
    *(a + x) = *(a + y);
    *(a + y) = temp;
}

int qsort(int* a, int l, int r)
{
    int i = 0;
    int last = 0;
    if(l >= r)
    {
        ret 0;
    }
    swap(a, l, (l + r) / 2);
    last = l;
    i = l + 1;
    while(i <= r)
    {
        if(*(a + i) < *(a + l))
        {
            last = last + 1;
            swap(a, last, i);
        }
        i = i + 1;
    }
    swap(a, l, last);
    qsort(a, l, last - 1);
    qsort(a, last + 1, r);
}

int main()
{
    int a[] = { 9, 43, 1, 5, 32 };
    qsort(a, 0, @a - 1); # The `@` operator returns the size of the array.
}
```
May you all have a good day:

`gcc -E qsort.c`
```
int main(void) {
 int fs[4096] = { 0 };
 int bs[4096] = { 0 };
 int vs[65536] = { 0 };
 register int fp = 1;
 register int fa = 0;
 register int sp = 0;
 register int rr = 0;
 goto main;
begin:
 if(fp == 0)
  return vs[sp - 1];
 switch(fa)
 {
swap:
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 2 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[vs[sp - 2]] = vs[sp - 1];;
 vs[sp - 2] = vs[sp - 1]; --sp;;
 --sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 2 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[vs[sp - 2]] = vs[sp - 1];;
 vs[sp - 2] = vs[sp - 1]; --sp;;
 --sp;
 --sp;
 --sp;
 --sp;
 --sp;
 vs[sp++] = 0;
 rr = vs[sp - 1];--fp;sp = bs[fp];fa = fs[fp];vs[sp] = rr;sp++;goto begin;
qsort:
 vs[sp++] = 0;
 vs[sp++] = 0;
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 2 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] = vs[sp - 2] >= vs[sp - 1]; --sp;
 if(vs[--sp] == 0) goto L0;
 vs[sp++] = 0;
 rr = vs[sp - 1];--fp;sp = bs[fp];fa = fs[fp];vs[sp] = rr;sp++;goto begin;
 goto L1;
L0:
L1:
 bs[fp] = sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 2 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp++] = 2;
 vs[sp - 2] /= vs[sp - 1]; --sp;
 fs[fp] = 110;fp++;goto swap;case 110:;
 --sp;
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[vs[sp - 2]] = vs[sp - 1];;
 vs[sp - 2] = vs[sp - 1]; --sp;;
 --sp;
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1;
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[vs[sp - 2]] = vs[sp - 1];;
 vs[sp - 2] = vs[sp - 1]; --sp;;
 --sp;
L2:
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 2 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] = vs[sp - 2] <= vs[sp - 1]; --sp;
 if(vs[--sp] == 0) goto L3;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp - 2] = vs[sp - 2] < vs[sp - 1]; --sp;
 if(vs[--sp] == 0) goto L4;
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1;
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[vs[sp - 2]] = vs[sp - 1];;
 vs[sp - 2] = vs[sp - 1]; --sp;;
 --sp;
 bs[fp] = sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 fs[fp] = 162;fp++;goto swap;case 162:;
 --sp;
 goto L5;
L4:
L5:
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp++] = 3 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1;
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[vs[sp - 2]] = vs[sp - 1];;
 vs[sp - 2] = vs[sp - 1]; --sp;;
 --sp;
 goto L2;
L3:
 bs[fp] = sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 fs[fp] = 184;fp++;goto swap;case 184:;
 --sp;
 bs[fp] = sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1;
 vs[sp - 2] -= vs[sp - 1]; --sp;
 fs[fp] = 195;fp++;goto qsort;case 195:;
 --sp;
 bs[fp] = sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 4 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 vs[sp++] = 1;
 vs[sp - 2] += vs[sp - 1]; --sp;
 vs[sp++] = 2 + bs[fp - 1];
 vs[sp - 1] = vs[vs[sp - 1]];
 fs[fp] = 206;fp++;goto qsort;case 206:;
 --sp;
 --sp;
 --sp;
 --sp;
 --sp;
 --sp;
 vs[sp++] = 0;
 rr = vs[sp - 1];--fp;sp = bs[fp];fa = fs[fp];vs[sp] = rr;sp++;goto begin;
main:
 vs[sp++] = 9;
 vs[sp++] = 43;
 vs[sp++] = 1;
 vs[sp++] = 5;
 vs[sp++] = 32;
 bs[fp] = sp;
 vs[sp++] = 0 + bs[fp - 1];
 vs[sp++] = 0;
 vs[sp++] = 0 + bs[fp - 1];
 --sp;
 vs[sp++] = 5;
 vs[sp++] = 1;
 vs[sp - 2] -= vs[sp - 1]; --sp;
 fs[fp] = 229;fp++;goto qsort;case 229:;
 --sp;
 --sp;
 vs[sp++] = 0;
 rr = vs[sp - 1];--fp;sp = bs[fp];fa = fs[fp];vs[sp] = rr;sp++;goto begin;
 }
 return 0;
}
```
