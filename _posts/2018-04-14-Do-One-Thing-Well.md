---
layout: post
---

Functions in C excel when transforming or creating one piece of data.

Have a look at typical Real Time Strategy game unit structure:

    typedef struct
    {
        int attack;
        int defense;
        int health;
    }
    Unit;

Constructing a new unit implies creating a stack unit.

    void xunew()
    {
        Unit u;
        u.attack = 7;
        u.defense = 3;
        u.health = 100;
        return u;
    }

Modifying the unit requires a function taking a single input and returning a single output.

    Unit xumodify(Unit u)
    {
        u.attack *= 2;
        return u;
    }

Any number of arguments thereafter will remain const:

    Unit xumodify(Unit u, const Map m, const Buildings bs, const Weather wr)
    {
        ...
        return u;
    }

Type objects like map, building, and weather, are not passed as const pointers
since the const can easily be dropped and their state modified. Sure, these types can
hold pointers within which can modify they data the point to even when the type is const,
but the onus on doing one thing well remains - only the unit type will be modified.

Have many inputs, but transform one piece of data. Make your functions do one thing well.
