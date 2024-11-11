---
layout: post
---

<iframe width="720" height="405" src="https://www.youtube.com/embed/v=jX-kl0I_fHk" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Directed Cyclic Graphs (DCG), when traversed breadth first, happen to model internal combustion fluid sim ordering. Seen above
is an inline 9 engine with plenum intake left, and four exhaust system, ejecting to the atmosphere right, with a turbo
charger making use of latent heat and pressure at the joint collector.

While this front end is not connected to the audio generator (just yet) in this post I want to outline that a DCG can,
in breadth first order, operate node-to-node in a perfectly parallel order, allowing no piston manifold channel to execute
its fluid or thermodynamic modelling out of order.

With a widget base class, each node is connected together by user input. Shared or weak pointers are automatically deduced,
and each node can at runtime be swapped for static volume (plenum, runner, exhaust, etc), and/or dynamic volume (a piston).
The conventional flow math, directed by isentropic choked flow, moves gas mass left to right, from widget to widget.

A node (as a parent) holds shared or weak children (next):

```
struct node_t : enable_shared_from_this<node_t>
{
    unique_ptr<widget_t> widget = {};
    list<variant<shared_ptr<node_t>, weak_ptr<node_t>>> next = {};
}
```

BFS iteration as a public member function serves to operate on the parent (finding a node, rendering a node, etc), and serves
to operate on parent to child (rendering lines between nodes, flowing from node to node, etc):

```
shared_ptr<node_t> node_t::iterate(
    const function<shared_ptr<node_t>(const shared_ptr<node_t>&)>& on_parent,
    const function<void(const shared_ptr<node_t>&, const shared_ptr<node_t>&, bool)>& on_parent_to_child)
{
    queue<shared_ptr<node_t>> q;
    q.push(shared_from_this());
    while(!q.empty())
    {
        shared_ptr<node_t> parent = q.front();
        q.pop();
        if(on_parent(parent))
        {
            return parent;
        }
        for(const variant<shared_ptr<node_t>, weak_ptr<node_t>>& node : parent->next)
        {
            bool is_cycling = false;
            shared_ptr<node_t> child = nullptr;
            if(const shared_ptr<node_t>* peek = get_if<shared_ptr<node_t>>(&node))
            {
                child = *peek;
                q.push(child);
            }
            else
            {
                is_cycling = true;
                child = get_if<weak_ptr<node_t>>(&node)->lock();
            }
            on_parent_to_child(parent, child, is_cycling);
        }
    }
    return nullptr;
}
```
