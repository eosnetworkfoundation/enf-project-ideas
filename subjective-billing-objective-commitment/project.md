# Subjective Billing with Objective Commitment

## Background
Currently on Mandel we bill subjectively.  This means that we poll the wallclock time for the amount of CPU used and then bill accordingly.

The issue with this is that it invariably sets up situations where we the node can be attacked with "speculative" bombs.

The brute force way around this is to go completely to objective billing.  This has its own set of upsides and downsides.

Objective billing can be good for creating solutions that don't allow for these bombs to take place as if the objective instruction count fails it fails the block.  But, it is hard to set these objective measures correctly and more importantly new hardware or nodeos optimizations won't impact the billed amount.

So, what we can do is do the work of objective billing and commit that to the block and still bill subjectively.

This will work in this manner:
   1. With a very low overhead solution, calculate the dynamic instruction retirement.
   2. Take that calculated objective count after the contract has finished and commit it to the block.
   3. On replay compare the committed objective count with the computed and if it starts to go over the computed then fail the block.

## Technical Details
Creating an optimized solution for this is paramount as we don't want to incur any noticable overheads.

### Low Overhead Instruction Counting
One key thing to take note of is that we only need to add to the instruction retirement counter at the head of a linear set of instructions.  This allows for the validation phase of nodeos to statically count these instructions and place one add operation at the head this linear grouping.  This means that at every control flow point we will have these adds.

If we find that the overheads are too expensive (lots of small linear chunks) we can coalesce these linear regions and add the max of these control flow paths.

For loops, the hope is that something like CDT will sufficiently unroll those loops so that a simple addition operation won't incur any noticable overhead.

This instruction adding will be done to a 'injected' global for WASM (or a out of range memory location for NatiVM).  We can validate that no code will try to interact with the injected global or memory location at set code time.  These will be satisfied by the invariants of the VMs.  We can then inject these simple additions to either this generated WASM global or memory location.

Because we are targeting x86_64 or even ARM64 in the future, both are modern superscalar architectures with good caches.  The counter location should be in cache and more than likely stay in the LSB during the running of the smart contract.

### Getting The Objective Measure
After we have finished execution of the action we will dip into the WASM module that was running and grab the global.

If we are replaying, this is a bit different as we need to check the result against the committed result frequently.
So, we will need to check these values frequently.  If the value is not updated yet then it is not that crucial of an issue. The only thing that matters is that we fail if committed instruction count < computed instruction count.