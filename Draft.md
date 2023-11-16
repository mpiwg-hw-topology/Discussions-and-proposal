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
(cf `MPI_Comm_idup`). This should be a collective call that does not necessarily
synchronize (probably not actually).

2- Init all communication operations

3- Wait for effective communication creation. Since the communication pattern is known
beforehand, optimizations can be applied (e.g ranks reordering). This can be achieved by
`MPI_Wait` or `MPI_Comm_commit`. The latter has to be collective and has to synchronize
(probalby). It does not just wait for the communicator creation but also
for the persistent operations to be set up and usable (as in prepared) as well.



### Issues/Open questions
~~1- Side-effect: lift some restriction on MPI_Comm_idup
(i.e. "It is erroneous to use the communicator newcomm as an input argument to other MPI
functions before the `MPI_COMM_IDUP` operation completes.")~~
=> Easy fix: specifiy in specs 

~~2- Pre-exisiting communications (`MPI_COMM_SELF`, `MPI_COMM_WORLD`) should be considered as
already committed and should not be committed again (no-op or erroneous if this
kind of handle is used in the commit procedure)~~
=> Easy fix: specifiy in specs  

3- Question: are the only operations authorized on the communicator the ones
registered before the call to wait/commit ?
=> TBD

~~4- Question: how to optimize with coalesced collective operations?~~
=> Easy fix: Not part of the proposal

5- Process identification issue: when/if creating a new communication (request) with
a call to a topology creation function, the reorder parameter can be set to 1.
Then the MPI process ranks will probably be different from the ones in the initial communicator.
The new ranks are supposed to be usable only once the new communicator is indeed available
(after the commit/wait operation). How can they be used in the calls to the procedures that initialize
the various persistent communication operations?
=> that's a tricky one, cf example V2


### Discussion
Based on the second example (`example_v2.c`):

1- Naming scheme creation functions are synchronizing because if/when reordering is
enabled, an MPI process might need the informations/parameters passed to the function
by other parameters. This is the case in particular for distributed graphs naming schemes
(a.k.a topology constructors) which might be synchronizing then (maybe not, depends
on how the mapping should be done). 
This might not be necessary in the Cartesian case, as parameters
are (should be) all the same on every involved MPI process.
There could be exchange of information (e.g. to gather hardware information) but
not necessarily with another MPI process but rather with a SW agent (that can even
be supported by a progress thread).

2- In the case of Cartesian topologies, since the structure is known, it is easy for
a process to identify its place in the topology. Its name a tuple (size = # of dimensions)
and a process can have a "good vision" in the topology of another process' place.
Also neighbors transitivity is possible because of the structure.
In the case of graphs and distributed graphs, little can be said about a process' "location"
in the topology once the naming scheme has been called/resolved. A process knows its
neighbors and the only thing usable is the reflexivity of the neighborhood relationship.

3- Notion of local addressing/identifiers, without a meaning outside each process that
determines the ordering of its neighbors. Reorder meaning in this case?? Myabe irrelevant.
So, it the naming is local, the procedure can be local. But if more information is needed
(e.g. values of parameters passed to the procedure by other processes), then synchronizing,
therefore non local.

4- Data partition should not be happening *before* the `MPI_Wait` operation that creates/commits
the new communicator. Issue: using the address of the data buffer before the processes know their
respective roles is not realistic. Maybe OK for collectives but not for pt2pt and RMA.
MPI_Send_init should be split into two parts.

How about an `MPI_Register_address` function ?
in the pt2pt op, call `MPI_BUFFER_DEFERRED` (a la `MPI_IN_PLACE`) instead of the buffer address.

Then after the `MPI_Wait` (trigerring the comm creation) call ` MPI_Register_buffer(void *addr,MPI_Request req);`
where req is the same handle that is ouptut from the init call (e.g. `MPI_Send_init`)
See [example_pt2pt.c](https://github.com/mpiwg-hw-topology/code-examples/blob/main/example_pt2pt.c).

OR:

an `MPI_Schedule_op` function which indicated which kind of operation is to be
performed on the communicator. Only the info pertaining to the communication pattern
is useful or is it not?


5- if topology is made restrictive, then the Bcast is similar in behaviour with pt2pt
operations w.r.t buffer addresses, plus some processes are not involved and call
`MPI_Bcast` anyway => unused address buffer. For such process, what is the meaning
of `MPI_Bcast_init`?





