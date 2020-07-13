---
layout: post
---

This post will outline a generic solution to lists, queues, and stacks (the latter two
simply being a simple rename of the first) in C99.

Other data structures (trees, hash maps, and the lot) will come at a later date.

## Doubly Linked Lists

The eternal memory allocator, malloc, being declared as:

    void* malloc(size_t size);

yields a plugin to a generic node datatype:

    typedef struct Node
    {
        void* data;
        struct Node* a;
        struct Node* b;
    }
    Node;

A node points to data allocated by malloc, and is doubly linked to data allocated by
malloc with both next (b) and previous (a) pointers.
(The naming is generic, as the Node data type will be reused for left and right
for trees in a later post).

Building a node reserves space for itself, and initializes data given by malloc:

    Node* Node_Init(void* data)
    {
        Node* node = malloc(sizeof(*node));
        node->data = data;
        node->b = NULL;
        node->a = NULL;
        return node;
    }

The doubly linked list points to the head and tail of the list. A destructor will define
the cleanup method for each node. This destructor callback is only used when the list is freed.
The size variable indicates the number of nodes in the list.

    typedef void (*Destruct)(void* data);

    typedef struct
    {
        Node* head;
        Node* tail;
        Destruct destruct;
        size_t size;
    }
    List;

    List* List_Init(Destruct destruct)
    {
        List* self = malloc(sizeof(*self));
        self->head = NULL;
        self->tail = NULL;
        self->size = 0;
        self->destruct = destruct;
        return self;
    }

Inserting a new node into the list can be done at the head or tail of the list. The End enum is superfluous,
as it may simply be replaced with a boolean, but its intent will make the later definitions of queues and
stacks a little more readable.

    typedef enum
    {
        HEAD, TAIL
    }
    End;

    void List_Push(List* self, void* data, End end)
    {
        Node* node = Node_Init(data);
        if(self->size == 0)
        {
            self->head = node;
            self->tail = node;
        }
        else
        {
            if(end == HEAD)
            {
                node->b = self->head;
                node->b->a = node;
                self->head = node;
            }
            if(end == TAIL)
            {
                node->a = self->tail;
                node->a->b = node;
                self->tail = node;
            }
        }
        self->size += 1;
    }

Popping an element from the list is done on a per node basis. This gives us the flexibility to
loop through and delete all nodes within in the list, or simply look up the head and tail nodes
and pop those. A destroy callback internally cleans up node data.

    void* List_Pop(List* self, Node* node, bool destroy)
    {
        Node* a = node->a;
        Node* b = node->b;
        void* data = node->data;
        if(node == self->head) self->head = b;
        if(node == self->tail) self->tail = a;
        if(a) a->b = b;
        if(b) b->a = a;
        if(destroy)
        {
            self->destruct(data);
            data = NULL;
        }
        free(node);
        self->size -= 1;
        return data;
    }

All operations on lists are done with loops. A single loop function with a _programmable_ callback
defines actions to be done on nodes. That is, the list iteration callback can stop or continue at some node,
or entirely delete the contents of some node.

    typedef enum
    {
        CONTINUE, STOP, DELETE
    }
    Action;

    typedef Action (*Iterate)(void* data, void* args);

The optional _args_ void pointer is used for further extending the functionality of the iteration callback:

    Node* List_For(List* self, Iterate iterate, void* args)
    {
        Node* node = self->head;
        while(node)
        {
            Node* b = node->b;
            switch(iterate(node->data, args))
            {
                case DELETE:
                    List_Pop(self, node, true);
                case CONTINUE:
                    break;
                case STOP:
                    return node;
            }
            node = b;
        }
        return NULL;
    }

Clearing the list pops the head until the size of the list is 0:

    void List_Clear(List* self)
    {
        while(self->size > 0)
            List_Pop(self, self->head, true);
    }

Freeing the list calls free, but ensures the list is cleared prior:

    void List_Free(List* self)
    {
        List_Clear(self);
        free(self);
    }

The doubly linked list may now be used in a generic setting, given _data_ is always
allocated by malloc (note free is passed as the destructor for free as no internal cleanup is required):

    List* list = List_Init(free);
    for(int i = 0; i < 42; i++)
    {
        int* temp = malloc(sizeof(*temp));
        *temp = i;
        List_Push(list, temp, TAIL);
    }

A printing function to print integers is defined as (note that CONTINUE is returned such that all elements are printed):

    Action Print(void* data, void* args)
    {
        (void) args;
        int* integer = data;
        printf("%d\n", *integer);
        return CONTINUE;
    }

And used as:

    List_For(list, Print, NULL);

Use of args is pretty much universal, but an example to sum the contents of a list is defined as:

    Action Sum(void* data, void* args)
    {
        int* integer = data;
        int* total = args;
        *total += *integer;
        return CONTINUE;
    }

And used as:

    int total = 0;
    List_For(list, Sum, &total);

Searching the list for elements that may exist is done similarly - STOP may be used to return the
first node found:

    Action Find(void* data, void* args)
    {
        int* integer = data;
        int* key = args;
        if(*integer == *key)
            return STOP;
        else 
            return CONTINUE;
    }

Used as:

    int key = 42;
    Node* data = List_For(list, Find, &key);
    if(data != NULL)
    {
        // FOUND.
    }

And of course, deleting all elements matching a key can be done with the above, but simply swap out

    return STOP;

for the alternate:

    return DELETE;

The above doubly linked list example used integers, but any object allocated by malloc can be used.
An employee, for instance, marked by name, salary, and an Orewellian ID can just be easily inserted
into a new list, searched for by name, deleted, printed, and so on:

    typedef struct
    {
        char* name;
        int salary;
        int id;
    }
    Person;

    Person* Person_Init(char* name, int salary, int id)
    {
        Person* person = malloc(sizeof(*person));
        person->name = name;
        person->salary = salary;
        person->id = id;
        return person;
    }

    Action Find(void* data, void* args)
    {
        Person* person = data;
        if(strcmp(person->name, args) == 0)
            return STOP; 
        else 
            return CONTINUE;
    }

    ...

    List* list = List_Init(free);
    List_Push(list, Person_Init("Paul Desmond", INT32_MAX, rand()), TAIL);

    ...

    Node* found = List_For(list, Find, "John Coltrane");

    ...

## Queues

The doubly linked list, being as flexible as it is, just needs a couple renames to build a queue.

    #define Queue_Init(destruct) List_Init(destruct)
    #define Queue_Free(self) List_Free(self)
    #define Queue_Enqueue(self, data) List_Push(self, data, TAIL)
    #define Queue_Dequeue(self) List_Pop(self, self->head, false)
    #define Queue_Peek(self) (self->head->data);

## Stacks

And of course, a stack is that of a queue, but popping (dequeues) are done from the same end as the pushing (enqueues):

    #define Stack_Init(destruct) List_Init(destruct)
    #define Stack_Free(self) List_Free(self)
    #define Stack_Push(self, data) List_Push(self, data, TAIL)
    #define Stack_Pop(self) List_Pop(self, self->tail, false)
    #define Stack_Peek(self) (self->tail->data)
