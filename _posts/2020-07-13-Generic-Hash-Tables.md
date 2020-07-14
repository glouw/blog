---
layout: post
---

Generic C99 hash tables (or dicts for the python programmers) can be readily built upon pre-existing generic list data structures.

The gist is that a generic portion of memory is hashed, and the hash retrieves a list from an array of lists (also known as buckets).
In the case of an insertion, the data is pushed to the back of this hashed list. In the case of a get, a node pointer is returned from this hashed list:

    typedef struct
    {
        List** list;
        Iterate iterate;
        size_t buckets;
    }
    Dict;

Initializing the dictionary requires an _iterate_ pointer be stored for matching the actual contents of the key with one of the items in the list.
The number of buckets is best suited by a prime number, roughly half the size of the maximum number of expected elements to be stored
such that hash collisions only need 2-3 list iterations for quick retrieval:

    Dict* Dict_Init(size_t buckets, Iterate iterate, Destruct destruct)
    {
        Dict* self = malloc(sizeof(*self));
        self->list = malloc(sizeof(*self->list) * buckets);
        for(size_t i = 0; i < buckets; i++)
            self->list[i] = List_Init(destruct);
        self->iterate = iterate;
        self->buckets = buckets;
        return self;
    }

The hashing is arbitrary - a key, of any data, with length is specified, and looped over, constructing a number that is
ultimately modulated by the number of buckets such that the look up index does not go out of bounds:

    unsigned Dict_Hash(Dict* self, void* key, size_t len)
    {
        char* byte = key;
        unsigned value = 0x0;
        for(size_t i = 0; i < len; i++)
        {
            value = (value << 4) + byte[i];
            unsigned temp = value & 0xF0000000;
            if(temp)
            {
                value = value ^ (temp >> 24);
                value = value ^ temp;
            }
        }
        return value % self->buckets;
    }

Retrieving a list, with a hash, is just an index:

    List* Dict_Index(Dict* self, void* key, size_t size)
    {
        size_t hash = Dict_Hash(self, key, size);
        return self->list[hash];
    }

Getting workable data from a dictionary just uses the index, and uses the _iterate_ callback to match
with the item in the indexed list:

    Node* Dict_Get(Dict* self, void* key, size_t size)
    {
        List* list = Dict_Index(self, key, size);
        return List_For(list, self->iterate, key);
    }

Inserting does a get to see if the data exists. If not, it inserts the data into the list.

    bool Dict_Insert(Dict* self, void* key, size_t size, void* data)
    {
        Node* found = Dict_Get(self, key, size);
        if(found)
            return false;
        else
        {
            List* list = Dict_Index(self, key, size);
            List_Push(list, data, TAIL);
            return true;
        }
    }

Unfortunately the call to Dict_index occurs twice in Dict_Insert, but is kept so for readability,
and is hopefully optimized away by the compiler.

Finally, freeing the dictionary is defined as:

    void Dict_Free(Dict** dict)
    {
        Dict* self = *dict;
        for(size_t i = 0; i < self->buckets; i++)
            List_Free(self->list[i]);
        free(self->list);
        free(self);
        *dict = NULL;
    }

Using a Person object, for example, hashes the persons name for its insert and get operations, and in
doing so, uses a function Person_MatchName for its _iteration_ callback which matches people by name:

    typedef struct
    {
        char* const name; // USED AS A KEY, SO REMAINS CONST.
        double age;
        double weight;
        double height;
    }
    Person;

    Person* Person_Init(char* name, double age, double weight, double height)
    {
        Person* self = malloc(sizeof(*self));
        *(char**) &self->name = name;
        self->age = age;
        self->weight = weight;
        self->height = height;
        return self;
    }

    void Person_Free(void* p)
    {
        Person* self = p;
        free(self);
    }

    Action Person_Print(void* data, void* args)
    {
        Person* self = data;
        char* prefix = args;
        if(prefix)
            printf("%s: ", prefix);
        printf("%s : %.2f %.2f %.2f\n", self->name, self->age, self->weight, self->height);
        return CONTINUE;
    }

    Action Person_MatchName(void* data, void* args)
    {
        Person* person = data;
        return strcmp(person->name, args) == 0 ? STOP : CONTINUE;
    }

Alas, a bucket size, being the prime number 1699, is used only as an example:

    int main(void)
    {
        Dict* dict = Dict_Init(1699, Person_MatchName, Person_Free);

        /* INSERT */;
        Person* person = Person_Init("Gustav Louw", 123, 456, 789);
        if(!Dict_Insert(dict, person->name, strlen(person->name), person))
            Person_Free(person);

        /* GET */
        char* name = "Gustav Louw";
        Node* found = Dict_Get(dict, name, strlen(name));
        if(found)
            Person_Print(found->data, "FOUND PERSON");

        Dict_Free(&dict);
    }

Any piece of data can be hashed, but the data must remain constant, else the hash will invalidate itself.
