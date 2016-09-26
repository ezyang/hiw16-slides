# GHCVM - A JVM Backend for GHCVM

Hi guys, I apologize for not being able to make to this year's ICFP. I
wanted to discuss about ways of making Haskell faster with the
opportunities the JVM provides, but I guess those discussions will have to
be saved for next year.

## Current State of GHCVM [1]

- Successfully compiles GHC 7.10.3 Haskell programs unmodified [1] as SPJ
requested during Frege Day 2015 [2].
- You can run them as well - a good class of programs will finish
successfully without blowing up on you.
- Cabal 1.22.8.0 was forked just last week and support was added for GHCVM.
You can build & run programs smoothly.
  - We call this CabalVM [3] and will probably submit patches upstream once
enough people use it and all the bugs are worked out.
  - I'm excited about Cabal 1.24.\*'s Nix-like package management and I will
patch that as well once it stabilizes.
- ghc-prim, integer are completely ported. The pure part of base is ported,
the part of base that relies on FFI still needs a lot of work in replacing
C FFIs with Java FFI calls.
  - Ptrs are implemented successfully using Java's DirectByteBuffer which
provides off-heap memory.
  - Brain McKenna did elementary work required to get `putStrLn` adn
`getLine` to work.
  - Still need to implement things like the IO Manager, POSIX shims, and
other OS-specific stuff
- You can connect to Java libraries - CabalVM supports a 'java-sources:'
field where you can include class files, jars, and java source files that
you want to be bundled into your library/executable.
  - The Java FFI is working. It relies on implementing a new type of
primitive - Object# c where c is a lifted type that is tagged with a string
that represents the Java class.
    - FFI is used in the implemention of the integer library and base
library.
    - Still working out how to keep handling of object primitives
type-safe, so far no need to deal with that because the FFI usage so far
has been on immutable Java objects.
      - The object primitives are equivalent to the mutable fields that was
recently proposed by Simon Marlow.
- The GHCVM-Hackage System [4] is a repository of patches for Hackage
libraries that CabalVM queries and patches to make them work with GHCVM.
  - This will allow users to directly install Hackage packages without
headache.
  - It's a temporary solution. We hope to maintain our own package server
at some point in the future and maintain some of the libraries ourselves.
  - Check out the README.md in the repo where I list out all the packages
from Hackage that currently work out of the box.
    - microlens, mtl, transformers are some examples

## GHC vs GHCVM
- Think of GHCVM's RTS as GHC's RTS written in Java, making sensible
translations where possible.
  - Because of the strong Java Memory Model, there are probably more
implicit memory fences sprinkled throughout compared to GHC.
  - The following are implemented (but not tested):
    - Pre-emptive scheduling
    - Asynchronous Exceptions
    - Exceptions
    - MVars
    - STM
    - StablePtrs
    - Lazy Blackholing
    - Nearly all other RTS primops except those relating to Sparks.
- Five different argument stacks are maintained:
  - Closures
  - Java Objects
  - Java ints
  - Java longs
  - Java doubles
  - Java floats
- The key difference is how the stack of update frames is handled.
  - In GHC, you return to the top of the stack by tail-calling to the
address at the top of stack.
  - In GHCVM, the stack is aligned with the Java call stack so that
returning from a closures' entry code method will return to the top of the
stack.
    - This hopefully allows the JVM to properly JIT closures' entry codes.
  - Disadvantage: STG stack is limited by JVM stack
    - Solution - stack chunking when stack exceeds a given limit. Save
stack frames and use Underflow Frame.
- All the primops necessary for the core Haskell libraries were implemented.
- No need to compile different runtimes for profiling, threaded, debug,
etc. The runtime will be customized via Java agents. Java agents are
plugins for the JVM that can modify bytecode at runtime.
  - To implement profiling in GHCVM, a Java agent will be implemented and
the bytecode will be generated to keep track of profiling info.
  - The threaded RTS can be activated by supplying a `+RTS --threaded` flag
to the program.

Future Plans for GHCVM:
- We won't be supporting GHC 8 anytime soon. We want the JVM people to get
used to basic types before wrapping their brains around dependent types.
  - But we will be cherry-picking bug fixes and features that work
independent of the type system changes in GHC 8.
- Implement optimizations
  - Currently all closure entries are slow calls via stacks. Making them
static method calls will yield a significant performance boost (can be done
in a day).
    - Done intentially to see the upper bound on running times of lazy
functional programs on the JVM.
  - Far future: Use GraalVM (in JDK 9) to boost performance.
    - Basic flow: serialize STG code and deseralize at the Graal level into
a Truffle AST and implement customized JIT optimizations specific for
Haskell.
- Support all the important of Hackage libraries (libraries will lots of
reverse dependencies)
- Re-use the external interpreter implementation from GHC 8 (via iserv) for
implementing GHCVMi and Template Haskell
  - We first need to port all the dependencies of iserv which will take
some time.

Commercial Plans for GHCVM:
TypeLead is a company I'm working on that will give commercial support for
GHCVM, provide a set of useful libraries for enterprise applications, and
provide online Haskell courses to teach Java programmers how to adapt their
existing Java knowledge to Haskell. It's still in ideation phase and my
co-founder (and my wife) got selected for the Startup Leadership Program
that gives us a great network and some other benefits (like cloud credits)
to work with. I think we have found a nice way to convey the practical
utility of Haskell to investors. We are currently raising funds to hire
amazing Haskellers around the world to build the world's best libraries for
the enterprise application development and evolve GHCVM at a rapid pace.
Our target is to close on funds by Summer of 2017, stabilizing GHCVM in the
meantime.

The idea is to take the stress off of GHC developers on catering to the
industry and continue letting it be a rapidly evolving research project. I
can see that the strain between industry and academia is intensifying in
the Haskell community in light of recent events.

Hope you enjoyed the summary. For more information, you can join us on
Gitter [5].

[1] http://github.org/rahulmutt/ghcvm
[2] https://github.com/Frege/frege/wiki/Simon-Peyton-Jones-transcript,-Frege-Day-2015
[3] https://github.com/rahulmutt/cabalvm/tree/1.22
[4] https://github.com/rahulmutt/ghcvm-hackage
[5] https://gitter.im/rahulmutt/ghcvm
