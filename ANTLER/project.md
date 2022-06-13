# ANTLER
## ANother TransLator Environment and Runtime
<img src="antler.png" alt="logo" width="300">

EOSIO.CDT is a tough system for me to look at. I had originally created the project with a few central tenets, that through too many cooks in the kitchen situations and optimizing for a singular person were abandoned relatively early.
Those tenets were to create a toolchain and environment for smart contracts developers that was robust, professional, performant, safe, easy to upgrade and easy to extend.

From almost CDT's inception I had perceived that C++ as the fundamental "smart contract" language was the wrong choice. As a systems language on the block chain (system contract, dapp utilities, etc.) is an okay choice. I will go into greater detail in a later section about this.

Sadly for close to four years I've watched my project not flourish as I had wanted. This is why I am going to propose a complete alternative to CDT and a lot of the other alternatives that have popped up over the last couple of years, with a lot of reasons as to why the proposed solutions should exist.

Motivations For A New Project
When building toolchains and tooling a lot of thought should go into certain areas of it, and in almost all solutions so far these concerns are not mentioned.

### Concerns
 - Does this solution produce binaries that are optimal (CPU/Size/Ram)?
 - Is this solution hardened of bugs?
 - Does this solution try to maintain safety as the default?
 - Does this solution add anything different to the current ecosystem?
 - Does the solution audit and prune library functions/classes or language constructs that are broken/impractical/unusable?

I focused on compilers and computer architecture for hyper embedded environments during my PhD work at Virginia Tech. So, when I say that smart contract development across the industry is analogous to embedded development, I absolutely mean it. The parallels exist at almost all layers, performance concerns, size concerns, requirements for specialized solutions for debugging/profiling/testing.

Most toolchain projects I have come across don't seem to have any focus on how performant or space efficient the tool will be. This is a worrying thing to me. The sentiment that these solutions will just cost more for CPU or for RAM is not a great argument. Not only should we be striving to lower the CPU costs and RAM costs consistently, this isn't the only issue. There is a very real upper bound to those things at which point certain types of smart contracts cannot be built at all.

There is a general laissez-faire attitude towards tooling and toolchains for smart contract development. This is deeply concerning to me as most of the smart contracts that will be used a lot will ultimately have millions/billions of dollars flowing through them. Bugs that cause funds to be lost are very real. This is especially true of new/untested tooling. The attitudes that I have continually run across for 4 years is that of "that compiles to WASM let's use that" I think are misguided. A modern highly optimizing compiler is not a trivial thing to produce and just because something compiles from language X to target Y doesn't necessarily mean that it will function well or be practical for use.

Some previous alternatives have pushed things like the most recent version of LLVM/Clang over what was in CDT. This type of mentality is also concerning as the main reason that CDT stopped at version 9 was that 10 didn't add anything of use and 11 and 12 caused extreme bloat and performance issues.

## Project Overview
A few big differences with this project and CDT need a specific call out.
The toolchain will be abstracted to allow for other devs to create frontends that attach into Antler's tooling.
A strong movement away from the way we have done ABI's to utilization of Google ProtoBufs.
Emphasis on ecosystem of tooling, so debugger, profiler, other ancillary tooling should be immediately available for those that build there language frontends against this solution.

## Base Layer
A new toolchain for C, C++, Rust and Go with associated pruned standard libraries. This tool will be called antler-comp.
A convention based build system called antler-build.
A convention based project/package management solution that utilizes antler-build, this will be called antler-pack.
Second Layer
Packages for EOS chain development will be made available. Along with packages for safe and secure smart contract construction. And other packages for quality of life.
A debugging framework that allows for debugging of smart contracts that have already been set on a running blockchain (I'm calling side-car debugging).
A testing framework for proper unit tests, integration tests, and system tests.

## antler-comp
This solution will come with C, C++, Rust and Go as first class citizens of the smart contract environment.
On top of this a unique layer will be exposed for other toolchain projects to adopt into this system. There will be four ways of creating your new language compiler.
 1. Directly generate LLVM-IR from your frontend. This is obviously the most efficient but requires a lot of domain specific knowledge.
 2. An RPC API will be provided to construct what I am calling a decorated AST. This will be the most preferred method as this API will hopefully not change much between versions. The reason for making this layer a set of RPC endpoints is that it will allow the devs creating the frontends to utilize whatever languages/technologies they want.
 3. This layer will be a bit of a stop gap as it will allow for the input of WASM.
 4. And lastly it will take x86–64 as input. Meaning that if you find a compiler that produces a shared object with debug info then you can feed it into this and produce the correct target code (WASM, NatiVM).

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