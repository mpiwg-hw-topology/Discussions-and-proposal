# Topologies (Draft)
--------------------

## Generic considerations

In the current version on the standard, (virtual topologies) are not directly
usable.
Topology *information* is attached to a communicator and is accessible
through a set of procedures.
Topologies officially come in three types:
1. cartesian topologies (`MPI_CART`)
2. general graph topologies (`MPI_GRAPH`)
3. distributed graphs topologies (`MPI_DIST_GRAPH`). Distributed graph topologies
were introduced to alleviate the scalability issues that exist in the
(earlier) non distributed version.

The topology type can be queried with a call to `MPI_TOPO_TEST(comm,status)` procedure
(status can also be `MPI_UNDEFINED`)

Two other types of virtual topologies are also more or less officially supported:

4. bipartite graphs through the properties of intercommunicators
5. neighborhood graphs: actually subset of general graphs (because of the
symmetric adjacency matrix requirement)


- **Question** Should these last two types be added to the previous list explicitly?
- **Question** Remove `MPI_CART`? (covered by `MPI_DIST_GRAPH`)
- Generalize Cartesian to euclidian space (support of diagonals)
- Elliptic topologies?

A Topology is rather a *topological* space. That is especially true for MPI_DIST_GRAPH
and more precisely MPI_CART is a *metric* space. 

If redesign of the interface: should the new objects align with their mathematical
counterparts? (b/c MPI is aimed at scientists).

This redesign is also the opportunity to switch from a flat programming model
to a more structured one, featuring neigborhoods and/or hierarchies for instance.

## Proposal

Promote topologies from implicit information attached to communicators to
full-fledged objects in the standard: `MPI_Topology`

### Properties/characteristics of the object

Instead of being *advisory* such topology object should be:
- **mandatory** (for objects such as communicators, windows, files and even groups
or process sets? see below).
Consequences:
	- MPI_UNDEFINED is no longer a valid status value after a call to `MPI_TOPO_TEST`.
	- A **default topology** is needed (in the World Process Model) that should be of type
	general graph or a distributed graph with full connectivity (to account for MPI flat
	programming model) , in case no topology is specified explicitely.
	- An interface has to be introduced, e.g. `MPI_Topo_attach(char *pset_name)` (see below) 

- **restrictive**: a topology describes a structure and MPI communication operations
must comply to it. This would, for instance, eliminate the need for neighborhood
collective operations (vs. "regular" collective operations, cf. Traff and al paper
about collective communication and orthogonality)

- **explicit**: a set of procedures must be added in order to access/query/manipulate
topologies. This would go beyond the set of current operations that mainly
work on the naming scheme/coordinates of the MPI processes in the topology.
**Question**: what is the set of operations we want to support?     	

- **nature**: Hardware vs. software (virtual)

### Relationship between topologies and other MPI objects

A possible way would be to adopt the same  scheme as in Sessions :
Process set -> group -> communicator

Here,  we start with a process set, an unstructured "object", with almost
no properties (a pset has a name and that's it) and turn it into something more structured.
For instance :

```mermaid
graph TD
      [MPI process set (unstructured)] + [MPI topology (a structure)]-->[an MPI Group];
      
```
We can then apply the same logic to create other MPI objects/concepts:
- MPI Group + context ID = an MPI Communicator (with a Topology since the group had one)
And then:
- MPI Comm + memory region = an MPI Window (now featuring a topology structure to boot)
- MPI Comm + File handle = MPI file (now featuring a topology structure to boot)


### Naming schemes/coordinates

- Topologies could also possess a **naming scheme** so that any MPI process can know
its location in the structure. Default topology's naming scheme : linear, 1-dimension one.

- **Question** What about distance functions for topologies? What is the meaning of
hops/neighbours/weights (ok for cartesian ones, but for general one ...)

- Naming schemes should also be promoted so that the user can efficiently
leverage them. Currently, it's possible to handle them only through:
	 - specific procedure parameters (e.g. the `reorder` parameter in `MPI_Comm_split`,
	 or `MPI_Cart_create`, etc. )
	 - specific procedures (e.g. `MPI_Cart_shift`, `MPI_cart_rank`, etc. )

- Naming schemes could be functions that replace the *reorder* parameter throughout
the standard.
    * pros:
	- user can provide their own naming schemes (right now, reordering
  works as a black-box without any user control over it).
  	- possibility to inject hardware information into the naming scheme by
  the MPI implementers but the application developers too.
  * cons:
    - breaks in the current standard. 

- Translation from one scheme to another:
Actually, reordering ranks is switching from a naming scheme to another.
Instead of a simple function parameter, can we have:
   `MPI_Reorder(IN int *tuple, OUT int* tuple);`
**Note**  This is more a mapping function actually  and Transformation != map
Functionality:

1- Transform a topology into another one or create a new topology from an existing one
(or even a set of input topologies into a new one)
2- Map: determine the coordinates of a Process in the new topology given the
coordinates in the old topology

**Question** Does reordering implies or necessitates isomorphism?


How is this different than calling N times `MPI_Graph_map`?

**Note**
- mapping a virtual topology onto another virtual topology => reordering of ranks
- mapping a virtual topology onto a hardware one => assignment of resources
! result might be different in both cases !

- But then we need another function to enforce this new binding (e.g. `MPI_Bind`)?


### Hardware topologies support

- Considered as explicit,
e.g. another type of `MPI_Topology` objects? Therefore:
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

- OTH, they could remain implicit, e.g. used as input for a translation/mapping
function between naming schemes.

### Process sets vs. Resource sets
- **Process sets** contain *processes* with virtual topology
A process set cannot be larger that the set named mpi://WORLD

- **Resource sets** contain *processors* with physical topology
A resource set can expand in the future of the application, especially
if/when MPI processes can migrate/change their resource binding


* map one topo to the other -> graph embedding according to some metric (bissection BW)
Map one of many virtual topologies to the hw one

Map -> did it do something?
    -> if yes, how well did it perform?


MPI_Topology

MPI_Topo_embedd(topo1, topo2, int *mapping_res, flag, quality)

Test of virtual topo types against the hw one and get a the quality result
to make the decision on which one to use.

How to pass user information (pattern, frequency, etc)?

## Possible new design

### Principle
The current way to use communicators with or without a topology attached to it is to:
1- create the communicator

2- create requests for persistent communications or call communication procedures
that use this communicator
When the communicator is created, it can't be optimized for the communication pattern
that is goind to be applied.

The proposed scheme reverses this state of things:
1- Create a request to construct a communicator (with or without topology information attached)
(cf `MPI_Comm_idup`)

2- Init all communication operations

3- Wait for effective communication creation. Since the communication pattern is known
beforehand, optimizations can be applied.

### Issues
1- Side-effect: lift some restriction on MPI_Comm_idup
(i.e. "It is erroneous to use the communicator newcomm as an input argument to other MPI
functions before the `MPI_COMM_IDUP` operation completes.")

2- Process identification issue: when/if creating a new communication (request) with
a call to a topology creation function, the reorder parameter can be set to 1.
Then the MPI process ranks will probably be different from the ones in the initial communicator.
The new ranks are supposed to be usable only once the new communicator is indeed available
(after the commit/wait operation). How can they be used in the calls to the procedures that initialize
the various persistent communication operations?





