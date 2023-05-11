Topologies

In the current version on the standard, (virtual topologies) are not directly
usable. Topology *information* is attached to a communicator and is accessible
through a set of procedures. Topologies come in three flavors: cartesian,
random graphs and distributed random graphs. Distributed graphs were introduced
to alleviate the scalability issues that exist in the earlier, non distributed
version. One could also consider that there is another kind of virtual topology
supported in the MPI standard: bipartite graphs, through the properties of
intercommunicators.

** Proposal :
* 1- Promote topologies from implict information attached to communicators to full-fledged 
objects in the standard. Also, instead of being advisory, topologies should be:
- mandatory (for objects such as communicators, windows, files and even groups
or process sets?)
- restrictive: a topology describes a structure and MPI communication opertions
must comply to it. This would, for instance, eliminiate the need for neighborhood
collective operations (vs. "regular" collective operations, cf. Traff and al paper
about collective communication and orthogonality)
- explicit: a set of procedures could be added in order to access/query/manipulate
topologies. This would go beyond the set of current operations that mainly
work on the naming scheme/coordinates of the MPI processes in the topology.

They also must possess a *naming scheme* so that any MPI process can know
who/where it is in the structure.

What about distance functions for topologies? What is the meaning of
hops/neighbours/weights (ok for cartesian ones, but for random one ...)

* 2- A possible way would be to adopt the same  scheme as in Sessions :
process set -> group -> communicator

Here,  we start with a process set, an unstructured "object", with almost
no properties (a pset has a name and that's pretty much all) and turn it into
something more structured. For instance :
      MPI process set (unstructured) + MPI topology (a structure)
                                     |
                                     V
                              an MPI Group

Since Topologies are mandatory, a default one should be provided in case none
is specified explicitely. This default topology must support all-to-all
communication pattern since it constitutes MPI's default, flat programming model.
Its naming scheme should be a linear, 1-dimension one.

We can even apply the same logic to create more MPI objects/concepts:
- MPI Group + context ID = an MPI Communicator (with a Topology since the group had one)
- MPI Group + memory resource = an MPI Window (now featuring a topology structure)
- MPI Group + File hande = MPI file (now featuring a topology structure)


* 3- Naming schemes should also be promoted so that the user can efficiently
leverage them. Currently, it's possible to handle them only through:
- specific procedure parameters (e.g. the *reorder* parameter in MPI_Comm_split, or MPI_Cart_create, etc)
- specific procedures (e.g. MPI_Cart_shift, MPI_cart_rank, etc )

Naming schemes could be functions that replace the *reorder* parameter throughout
the standard.
* pros:
  - user can provide their own naming schemes (right now, reordering
  works as a black-box without any user control over it).
  - possibility to inject hardware information into the naming scheme by
  the MPI implementers but the application developers too.
* cons:
  - breaks in the current standard. 

Translation from one scheme to another:
Actually, reordering ranks is switching from a naming scheme to another.
Instead of a simple function parameter, can we have:
   MPI_Reorder(IN int *tuple, OUT int* tuple);

How is this different than calling N times MPI_Graph_map?

Remark:
- mapping a virtual topology onto another virtual topology => reordering of ranks
- mapping a virtual topology onto a hardware one => allocation of resources

But then we need another function to enforce this new binding (MPI_Bind)?


* 4- Hardware topologies?
How do we want to support HW topologies?
- They could be considered as explicit,
e.g. another type of MPI_Topo objects. We then would have something like:

MPI_Topology object:
   - Nature : virtual vs hardware
                 |           |
                 v           v
              - cartesian   - random
              - dist graph  - fat-tree
              - bipartite   - dragonfly
              - else?       - ?

Do we impose not one but two topologies for each pset/group/comm
(a virtual + a physical one)?

- They could remain implicit, e.g. used as input for a translation/mapping
function between naming schemes.

