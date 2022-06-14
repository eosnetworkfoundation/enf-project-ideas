# Parallelism, Parallelism, and More Parallelism

## Why do we need parallelism?
Currently, Mandel is designed for a timeline that didn't occur.  As in; single-threaded heavy high clock speed processors. This isn't the path that CPU companies took.  They took the path of more hardware threads and relatively low clock speeds.  This leads to Mandel/EOSIO has poor utilization of the underlying system and requires very expensive hardware to be used.  This is not good for a number of reasons: too expensive for others to join into, not very scalable, at some point we will reach the limitations of the current hardware altogether.

Personally, I believe that scalability starts with parallelism at the smart contract evaluation level and at the node level.  The issue that I have seen with other proposals for scalability that are achieved with IBC solutions or by fracturing a chain is that they open more doors than they close.

I believe this model is heavily proposed because other chains like Ethereum have deemed that to be the way that they achieve scalability.  The issue is that Ethereum is not EOSIO/Mandel and they have grossly different invariants. Ethereum is designed to run on any hardware available, also because of their consensus mechanism they can't really optimize for good hardware to be the rule.  Because we have a vastly different environment, being able to optimize for scalability at the node level is a much different beast.

With IBC you need to concern yourself with how to fracture that mono-state of a chain and not replicate the data across all of the sub-chains, how do you synchronize between them, and how to handle these things at a cryptographically provable way.  This means expensive and hard to architect.  IBC as a solution for inter-blockchain communication and message passing is sorely needed, but as a solution to scalability, I think it is a bit of a fool's dream.

## Asynchronous DB
The first area of interest for parallelism is around our underlying database for `multi-index` and any future database backends we support should allow for the same interfaces.

Currently, chainbase is a single-threaded solution.  This is not incredibly ideal as we waste time by not being able to carry on with valuable computation until we get the results back or write out the data.

The first is asynchronous writes, as this allows us to take them off of the critical path of execution for smart contracts.

A write queue (much like a store queue in superscalar architectures) will be employed to buffer the writes out.

These writes don't need to occur until we finalize the transaction so there is plenty of time for these to be applied.

The next is asynchronous reads (we will discuss synchronizing in asynchronous actions and speculative parallelism).

By making reads asynchronous we can start to do a few things to gain performance:
   - Prefetch iterations, i.e. if we get an increment, then start to grab the next one, if we get a decrement then we start to grab the previous one.
   - In CDT/ANTLER we can reschedule these reads to occur as soon as we have sufficient data to call the function and optimize for being able to do valuable work before the result is required.  This is analogous to compilers rescheduling load instructions as these can be evaluated in parallel at the ILP level.
   - Cluster reads, i.e. read one and then start the process of reading quite a few more.  By the time the first result is done, you will more than likely have the results for the rest then or shortly after.


The result of the initial call will be an opaque handle to the promised data.  A separate host function will be available to get the result.

Rough examples:
```c
int64_t get_row(...);
int64_t get_data_size(int64_t handle);
int get_data(int64_t handle, char* buff, uint32_t size);
```

Where `get_row` sets off the asynchronous read, `get_data_size` checks if the data is ready and blocks if it isn't and returns either negative values for errors or the size of the data, and finally, `get_data` will retrieve the data and return a possible error code.

## Asynchronous Actions
As mentioned in ANTLER these can be incredibly useful.

The API for asynchronous actions is the same kind of API as the async database above.

For the first version of async actions we will only operate on `read-only` actions or what I will refer to as `pure` actions.  These have no side effects to the state of the system.

By enforcing that only `pure` actions are available for asynchronous execution we can mitigate the need for strong synchronization between actions.

Later we can support non-pure actions.

By using a similar API as above:
```c
int64_t async_send(...);
int64_t get_action_result_size(int64_t handle);
int get_action_result(int64_t handle, char* buff, uint32_t size);
```

To overcome the performance issues with calling into other actions we will implement a double buffer for the linear memory for each thread of execution.  This will ensure that each action run on a thread should be able to immediately load into memory and not have to clean out its state first and reset memory protections.

Also, the addition of another host function:
```c
void prefetch_contract(name contract, bool discard);
void discard_contract(name contract);
```

This will allow you to prefetch a contract that will be executed to a thread and the `discard` flag will specify whether the thread throws out the state of that contract.

This shouldn't really be used directly as the compiler will add these prefetches at the most efficient spots in the execution flow to minimize cost and not cause issues with too many prefetched contracts.

## Speculative Paralellism
Being able to parallelize regular actions will allow for Mandel to lower CPU costs by N times and get better system utilization for existing server grade hardware.

The key takeaways from this type of parallelism is that we will speculatively run an action in parallel and if a data-hazard is hit then we kill that execution and reschedule it for the "owning" thread of that data.

A map of table rows that have been written to will be kept with the value of the owning thread.

Key: table row modified
Value: owning thread (first to write)

During execution when a DB table row is read it will check the monitor to see if anyone owns it, if not it will read the from the DB if someone does own it it will kill the execution and reschedule to the owning thread.

If during execution a DB table row is written to it will check the monitor to see if anyone owns it, if not it will claim ownership and write its data, if someone does own it it will kill the execution and reschedule to the owning thread.

This ensures that we don't necessarily need to worry about fine grained synchonization and how to encode that for replay.

Transactions with good variation of actions will be the most successful in this solution.

Also, mono-state smart contracts will not get a lot of benefit from this solution. Meaning, things like simple counters, singletons, etc. will force serialization.  There are solutions around this with state fracturing patterns that can be abstracted in things like CDT and as best practices patterns.

To get the most out of replay we will want to commit the tree of execution scheduling to each block.

There will also be a way of defining ordering of actions if dependencies are known at the point of transaction creation.

## Distributed Node
By building on to the ideas of above we are finally led to parallelism at the node level.  This alleviates a lot of the scalability concerns of massive database sizes.

By fracturing the database state we can lower RAM pressure on each node and we can then employ the same "speculative" model over the concept of the transaction.

This would extend the map to a synchronized map to all nodes and will list ownership of either local or external.

A new model for nodeos will be needed with the "lead" node being stateless that takes input transactions and queues them for redirection to the distributed nodes.

These distributed nodes will take these transactions and speculatively execute them and return transaction information for construction of the block.

Because we have the async DB, we can leverage the same API and concepts to optimize reads by hoisting them to as soon as possible.

If an async action in a transaction fails from DB ownership that transaction is failed and rescheduled to the owning node.

For replay performance the scheduling of the transactions would want to be committed to the block.

## Optimizations for speculative parallelism and distributed node
When we build the smart contract we can do some probabilistic analysis with the creation of "symbolic execution slices". These can then be leveraged when the transaction is queued.  By ducking the values through this probability function, we can try to attain better affinity for which thread to initially schedule to or node and find probable data hazards.
This does not need to be perfect, if the answer is wrong then it only hurts the smart contract and not the node or the network.