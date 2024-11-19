---
layout: post
---

<iframe width="720" height="405" src="https://www.youtube.com/embed/jX-kl0I_fHk" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Directed Cyclic Graphs (DCG), when traversed breadth first, happen to model internal combustion fluid sim ordering. Seen above
is an inline 8 engine with plenum intake left, and four exhaust system, ejecting to the atmosphere right, with a turbo
charger making use of latent heat and pressure of each exhaust system.

While this front end is not connected to the audio generator (just yet) in this post I want to outline that a DCG can,
in breadth first order, operate node-to-node in a perfectly parallel order, allowing no piston manifold channel to execute
its fluid or thermodynamic modelling out of order.

With a widget base class, each node is connected together by user input. Shared or weak pointers are automatically deduced,
and each node can at runtime be swapped for static volume (plenum, runner, exhaust, etc), and/or dynamic volume (a piston).
The conventional flow math, directed by isentropic choked flow, moves gas mass left to right, from widget to widget.

A node (as a parent) holds shared or weak children (next):

```
template <typename T>
using non_unique_ptr = variant<shared_ptr<T>, weak_ptr<T>>;

struct node_t : enable_shared_from_this<node_t>
{
    unique_ptr<widget_t> widget = {};
    list<non_unique_ptr<node_t>> next = {};
}
...
```

BFS iteration as a public member function serves to operate on the parent (finding a node, rendering a node, etc), and serves
to operate on parent to child (rendering lines between nodes, flowing from node to node, etc):

```
...
using handle_node = function<shared_ptr<node_t>(const shared_ptr<node_t>&)>;
using handle_edge = function<shared_ptr<node_t>(const shared_ptr<node_t>&, const shared_ptr<node_t>&, bool)>;

shared_ptr<node_t> node_t::iterate(const handle_node& handle_node, const handle_edge& handle_edge)
{
    queue<shared_ptr<node_t>> q;
    q.push(shared_from_this());
    while(!q.empty())
    {
        shared_ptr<node_t> parent = q.front();
        q.pop();
        if(handle_node(parent))
        {
            return parent;
        }
        for(const non_unique_ptr<node_t>& node : parent->nodes)
        {
            if(holds_alternative<shared_ptr<node_t>>(node))
            {
                shared_ptr<node_t> child = get<shared_ptr<node_t>>(node);
                q.push(child);
                handle_edge(parent, child, false);
            }
            else
            if(shared_ptr<node_t> child = get<weak_ptr<node_t>>(node).lock())
            {
                handle_edge(parent, child, true);
            }
        }
    }
    return nullptr;
}
...
```

Where utilization might be defined as:

```
...
graph->iterate(
    [](const std::shared_ptr<node_t>& parent)
    {
        parent->widget->do_work();
        return nullptr;
    },
    [](const std::shared_ptr<node_t>& parent, const std::shared_ptr<n
    {
        parent->widget->flow(child->volume);
        return nullptr;
    }
);
...
```

do_work() performs mechanical and elctrical work on a widget (adiabatic compression,
spark plug ignition), and flow() performs the transfer of mols, via isentropic flow,
from one node to the next.
