# ANTLER
## ANother TransLator Environment and Runtime
<img src="antler.png" alt="logo" width="300">

EOSIO.CDT is a tough system for me to look at. I had created the project with a few central tenets that were abandoned relatively early through too many cooks in the kitchen situations and optimizing for a singular person.
Those tenets were to create a toolchain and environment for smart contract developers that was robust, professional, performant, safe, easy to upgrade and easy to extend.

From almost CDT's inception, I had the realization that C++ as the fundamental "smart contract" language was the wrong choice. As a system language on the blockchain (system contract, dapp utilities, etc.) is an okay choice. I will go into greater detail in a later section about this.

Sadly, for nearly four years, I've watched my project not flourish as I had wanted. This is why I am going to propose a complete alternative to CDT and the other options that have popped up over the last couple of years, with many reasons as to why the proposed solutions should exist.

## Motivations For A New Project
When building toolchains and tooling, a lot of thought should go into some regions of it, and in almost all solutions, these concerns are not mentioned.

### Concerns
 - Does this solution produce optimal binaries (CPU/Size/Ram)?
 - Is this solution hardened against bugs?
 - Does this solution try to maintain safety as the default?
 - Does this solution add anything different to the current ecosystem?
 - Does the solution audit and prune library functions/classes or language constructs that are broken, impractical, or unusable?

I focused on compilers and computer architecture for hyper-embedded environments during my Ph.D. work at Virginia Tech. So, when I say that smart contract development across the industry is analogous to embedded development, I mean it. The parallels exist at almost all layers: performance concerns, size concerns, and requirements for specialized debugging/profiling/testing solutions.

Most toolchain projects I have come across don't seem to focus on how performant or space-efficient the tool will be. This is a worrying thing to me. The sentiment that these solutions will cost more for CPU or RAM is not a great argument. Not only should we strive to lower CPU and RAM costs consistently, but this isn't the only issue. There is a very real upper bound to those things, at which point certain types of smart contracts cannot be built at all.

There is a general laissez-faire attitude towards tooling and toolchains for smart contract development. This is deeply concerning to me as most of the smart contracts that will be used a lot will ultimately have millions/billions of dollars flowing through them. Bugs that cause funds to be lost are very real. This is especially true of new/untested tooling. The attitudes that I have continually run across for four years is that of "that compiles to WASM let's use that" I think are misguided. A modern, highly optimizing compiler is not a trivial thing to produce, and just because something compiles from language X to target Y doesn't necessarily mean that it will function well or be practical for use.

Some previous alternatives have pushed things like the most recent version of LLVM/Clang over what was in CDT. This mentality is also concerning and was the main reason CDT stopped at version 9 and not 10 as it didn't add anything of use, and 11 and 12 caused extreme bloat and performance issues.

## Project Overview
A few big differences between this project and CDT need a specific call-out.
The toolchain will be abstracted to allow other devs to create frontends that attach to Antler's tooling.
A strong movement away from the way we have done ABI's to utilization of Google ProtoBufs.
Emphasis on the tooling ecosystem, so debugger, profiler, and ancillary tooling should be immediately available for those who build their language frontends against this solution.

## Base Layer
A new C, C++, Rust, and Go toolchain with associated pruned standard libraries.  Strong languages abstractions and parity between the languages in terms of features and abilities will be there. This tool will be called antler-comp.
A convention-based build system is called antler-build.
A convention-based project/package management solution that utilizes antler-build will be called antler-pack.

## Second Layer
Packages for EOS chain development will be made available. Along with packages for safe and secure intelligent contract construction. And other packages for quality of life.
A debugging framework that allows for debugging of smart contracts in isolation and ones that have already been set on a running blockchain (I'm calling side-car debugging).
A testing framework for proper unit, integration, and system tests.

## antler-comp
This solution will come with C, C++, Rust and Go as first class citizens of the smart contract environment.
On top of this, a unique layer will be exposed for other toolchain projects to adopt this system. There will be four ways of creating your new language compiler.
 1. Directly generate LLVM-IR from your frontend. This is the most efficient but requires a lot of domain-specific knowledge.
 2. An RPC API will be provided to construct what I am calling a decorated AST. This will be the most preferred method as this API will hopefully not change much between versions. The reason for making this layer a set of RPC endpoints is that it will allow the devs creating the frontends to utilize whatever languages/technologies they want.
 3. This layer will be a bit of a stop gap as it will allow for the input of WASM.
And lastly, it will take x86–64 as input. If you find a compiler that produces a shared object with debug info, you can feed it into this and create the correct target code (WASM, NatiVM).

## antler-build
The current utilization of CMake is a less than great solution. It is an okay solution for developers who are already C++ developers but it has always been yet another thing for developers to learn. In addition CMake can be incredibly difficult to learn and use correctly.
So, in comes antler-build. This tool will use a very terse YAML based language and in conjunction with antler-pack package conventions allow for incredibly easy building of smart contracts and libraries.
Example `build.yaml`
```yaml
project: smart-contract
version: 1.0.0
components:
   - component: 
          name: foo
          type: contract
          lang: c
          depends: [eoslib, safemath, somebody:gamelib]
          comp_flags: [-DSOMETHING, -stack-canary]
          link_flags: [-stack-canary]
   - component:
          name: foolib
          type: library
          lang: go
          depends: [safemath, gamelib]
```

I would love to see more community interplay with packages and libraries and the like. This along with antler-pack should help to make that a reality.

## antler-pack
This will define the package conventions for layout of the project, construction of packages and pulling packages from github.
The package conventions are:
  - The root directory of the project should have the same name as the project name.
  - A series of internal directories will exist: (tests, src).
  - Inside of src subdirectories with the names of the components should exist. An include directory can be made here to hold header files that are to be used project wide.
  - Inside each of these subdirectories an include directory can be made.

### Why Choose C/C++/Rust/Go?
As mentioned above, I feel that C++ is not a great choice for general smart contract development.  This is primarily because it is a very low-level systems-level language.

Now the astute will say that C, Rust and Go are all systems-level languages, and this is true. However, I am angling for more options to choose from so that there is a greater chance of having more expertise in one of those languages.

Invariably I feel that application-level languages and scripting languages are not the greatest fit either.  But, would love to see all kinds of languages used for development as each one has its own benefits and downsides. 

I have a language that I have been designing for a while that I am calling `Omega` that is takes influence from Microsoft R&D's `CΩ`.  With the main focuses on stream processing abstractions, eventing, sequencing and safety.  A larger article will be available for that later and definitely not in scope for anything to be soon.

### What's This About No ABI's?
Another proposal I have is `asynchronous actions`.  A part of this proposal is `read-only` actions or what I am going to refer to as `pure` actions for ANTLER.

Once we have these `pure` actions, we can start to view the data interchange model as "everything is an action".

By moving to Google ProtoBufs we can buy into the preexisting environment of tooling and support for that serialization/deserialization format.  Currently any new language that wants to support Mandel has to implement the "FC" serialization/deserialization format and our ABI weird JSON format.  This is less than ideal as it stunts quick tooling from being produced.  This also allows any new languages that are going to be produced against this system to take off with a preexisiting protobuf compiler.

This does change the way data is accessed from the DB (persistent storage) of a smart contract.  Currently, any smart contract or outside tooling can read directly into these tables and do something with the data.  You would still be able to do that with the protobufs solution but you shouldn't assume anything other than it is a blob.

The way the DB API is currently handled is a horrible abstraction leakage.  There is no reason as why nodeos should know the internal serialization/deserialization.

By, moving to this model it allows smart contracts to define the APIs that they supply to other smart contracts and the outside world.

More on this model is in the proposal `protobuf ABIs`.