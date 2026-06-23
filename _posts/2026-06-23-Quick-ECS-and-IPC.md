---
layout: post
---

An Entity Compoenent System in C23 can generate _incredibly_ optimized SIMD
instructions on SOA entities while emphasizing maintenance with this X-Macro
appraoch:

```
#define ent_d(exec) \
    exec(float, x)  \
    exec(float, y)  \
    exec(float, z)  \
    exec(float, vx) \
    exec(float, vy) \
    exec(float, vz)

constexpr auto g_ents_cap = 8192;

struct
{
    #define exec(type, field) type field[g_ents_cap];
    ent_d(exec)
    #undef exec
}
g_ents = {};

auto g_ents_size = 0;

typedef struct
{
    #define exec(type, field) type field;
    ent_d(exec)
    #undef exec
}
ent_t;

void push(const ent_t ent)
{
    #define exec(type, field) g_ents.field[g_ents_size] = ent.field;
    ent_d(exec)
    #undef exec
    g_ents_size++;
}

void move()
{
    for(auto i = 0; i < g_ents_cap; i++)
    {
        g_ents.x[i] += g_ents.vx[i];
        g_ents.y[i] += g_ents.vy[i];
        g_ents.z[i] += g_ents.vz[i];
    }
}
```

The `move()` function operates on the constant capacity size `g_ents_cap`,
and not `g_ents_size` to eliminate SIMD tail branching:

```
00000000000011a0 <move>:
    11a0:   lea     0x2eb9(%rip),%rax
    11a7:   lea     0x8000(%rax),%rdx
    11ae:   xchg    %ax,%ax
    11b0:   vmovaps (%rax),%ymm0
    11b4:   add     $0x20,%rax
    11b8:   vaddps  0x17fe0(%rax),%ymm0,%ymm0
    11c0:   vmovaps %ymm0,-0x20(%rax)
    11c5:   vmovaps 0x7fe0(%rax),%ymm0
    11cd:   vaddps  0x1ffe0(%rax),%ymm0,%ymm0
    11d5:   vmovaps %ymm0,0x7fe0(%rax)
    11dd:   vmovaps 0xffe0(%rax),%ymm0
    11e5:   vaddps  0x27fe0(%rax),%ymm0,%ymm0
    11ed:   vmovaps %ymm0,0xffe0(%rax)
    11f5:   cmp     %rdx,%rax
    11f8:   jne     11b0 <move+0x10>
    11fa:   vzeroupper
    11fd:   ret
```

In practice whether this is faster than processing only `g_ents_size` is up for
debate, but the generated assembly is nicer.

### A Tale of IPC:

The SOA move has IPC of 1.5:
```
 Performance counter stats for './a.out 10000':

                 0      context-switches:u #    0.0 cs/sec  cs_per_second
                 0      cpu-migrations:u   #    0.0 migrations/sec  migrations_per_second
               633      page-faults:u      # 5215.8 faults/sec  page_faults_per_second
            121.36 msec task-clock:u       #    0.9 CPUs  CPUs_utilized
             8,793      branch-misses:u    #    0.0 %  branch_miss_rate         (49.63%)
        41,243,880      branches:u         #  339.8 M/sec  branch_frequency     (50.45%)
       401,636,074      cpu-cycles:u       #    3.3 GHz  cycles_frequency       (50.61%)
       607,399,083      instructions:u     #    1.5 instructions  insn_per_cycle  (50.37%)

    0.122078331 seconds time elapsed
```

Swapping the layout of `g_ents` from SOA to AOS improves IPC to 2.7:

```
 Performance counter stats for './a.out 10000':

                 0      context-switches:u #    0.0 cs/sec  cs_per_second
                 0      cpu-migrations:u   #    0.0 migrations/sec  migrations_per_second
               825      page-faults:u      # 1313.2 faults/sec  page_faults_per_second
            628.23 msec task-clock:u       #    1.0 CPUs  CPUs_utilized
            10,081      branch-misses:u    #    0.0 %  branch_miss_rate         (50.23%)
       163,270,204      branches:u         #  259.9 M/sec  branch_frequency     (50.13%)
     1,732,942,939      cpu-cycles:u       #    2.8 GHz  cycles_frequency       (49.99%)
     4,617,009,107      instructions:u     #    2.7 instructions  insn_per_cycle  (49.77%)

    0.631062773 seconds time elapsed
```

The hardware is certainly saturated with more busy work, but the cycle count and wall clock time is nearly 5x in size.

Inspecting the AOS move:

```
00000000000012e0 <move>:
    12e0:   lea       0x2d59(%rip),%rax
    12e7:   lea       0x180000(%rax),%rdx
    12ee:   xchg      %ax,%ax
    12f0:   vmovq     0x20(%rax),%xmm4
    12f5:   vmovq     0x8(%rax),%xmm2
    12fa:   add       $0x30,%rax
    12fe:   vmovq     -0x18(%rax),%xmm0
    1303:   vmovq     -0x20(%rax),%xmm3
    1308:   vinsertps $0x10,%xmm0,%xmm2,%xmm1
    130e:   vinsertps $0x40,%xmm3,%xmm4,%xmm5
    1314:   vmovlhps  %xmm4,%xmm0,%xmm0
    1318:   vmovq     -0x8(%rax),%xmm4
    131d:   vpermilps $0x99,%xmm0,%xmm0
    1323:   vmovlhps  %xmm3,%xmm2,%xmm2
    1327:   vmovq     -0x30(%rax),%xmm3
    132c:   vaddps    %xmm5,%xmm1,%xmm1
    1330:   vaddps    %xmm4,%xmm0,%xmm0
    1334:   vpermilps $0x99,%xmm2,%xmm2
    133a:   vaddps    %xmm3,%xmm2,%xmm2
    133e:   vmovss    %xmm1,-0x28(%rax)
    1343:   vmovshdup %xmm0,%xmm6
    1347:   vmovlhps  %xmm0,%xmm1,%xmm1
    134b:   vpermilps $0x99,%xmm1,%xmm1
    1351:   vmovlps   %xmm2,-0x30(%rax)
    1356:   vmovlps   %xmm1,-0x18(%rax)
    135b:   vmovss    %xmm6,-0x10(%rax)
    1360:   cmp       %rax,%rdx
    1363:   jne       12f0 <move+0x10>
    1365:   ret
```

More partial loads, more scalar ops, more register shuffles, more partial writes. IPC does not equal performance.
