# Topologies (Draft)
--------------------

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

Two other types of virtual topologies are also more or less offici allysupported:

4. bipartite graphs through the properties of intercommunicators
5. neighborhood graphs: actually subset of general graphs (because of the
symmetric adjacency matrix requirement)

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
process set -> group -> communicator

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
- MPI Comm + memory region = an MPI Window (now featuring a topology structure)
- MPI Comm + File handle = MPI file (now featuring a topology structure)


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

How is this different than calling N times MPI_Graph_map?

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
