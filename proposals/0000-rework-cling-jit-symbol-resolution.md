# Rework cling jit symbol resolution

* Proposal: [RE-NNNN](NNNN-rework-cling-jit-symbol-resolution.md)
* Authors: [Vassil Vassilev](https://github.com/vgvassilev)
* Review Manager: TBD
* Status: **Awaiting review**

<!--

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://root-forum.cern.ch/c/root-evolution), [Additional Commentary](https://root-forum.cern.ch/c/root-evolution)
* Bugs: [ROOT-NNNN](https://sft.its.cern.ch/jira/projects/ROOT/issues/ROOT-NNNN), [ROOT-MMMM](https://sft.its.cern.ch/jira/projects/ROOT/issues/ROOT-MMMM)
* Previous Revision: [1](https://github.com/root-project/root-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [RE-XXXX](XXXX-filename.md)

-->

## Introduction

Currently, cling's symbol resolution mechanism gives a preference to the local content to itself, i.e. resolves symbols from where its JIT resides (either its own binary or a shared library, libCling.so) first. This ensures that the symbol resolution will pick up compatible `llvm` and `clang` symbols for its runtime. However, this has a drawback: ROOT cannot provide its own low-level support libraries such as `zlib` other than the ones linked against libCling. 

<!-- TODO

root-evolution thread: [Discussion thread topic for that proposal](https://root-forum.cern.ch/c/root-evolution)

-->

## Motivation

ROOT provides its own `zlib` library if we choose `-Dbuiltin_zlib=On`. It has been optimized to yield much better performance (~4x according to recent presentations [citation needed]) than system `zlib`. If we enable this option LLVM and libCling will still be built against the system `zlib`. This introduces compile time and runtime mismatch potentially resulting in picking up the slower system symbols.

For instance:
```~cpp
[root] #include "zlib.h"
[root] zlibVersion()
(const char*) 1.2.3
[root] ZLIB_VERSION
(const char*) 1.2.4
```

This happens because cling's symbol resolution follows different rules than a regular linker which can lead to many unexpected situations. For example, not being able to find a globally overriden operator `new` (because LLVM provides one, too). This would mean that even if we provide an overload of `new` in `libCore`, cling will still find the symbols from llvm's library. This behavior is by design, because we want to resolve the embedded in ROOT LLVM symbols first. Otherwise cling will start picking up random LLVM symbols introducing a lot of compatibility and versioning issues.


## Proposed solution

Describe your solution to the problem. Provide examples and describe how they work. Show how your solution is better than current workarounds: is it cleaner, safer, or more efficient?

The existing workarounds are insufficient:
  * We cannot disable zlib for LLVM (as [PR1174: ROOT-8839 Disabling LLVM_ENABLE_ZLIB option for LLVM](https://github.com/root-project/root/pull/1174) suggests) because our PCH (and soon PCM) infrastructure embeds compressed header files ([PR893: Embed all header files (as zip) in the PCH](https://github.com/root-project/root/pull/893)). If we disable LLVM's zlib support ROOT's PCH becomes 15% larger.
  * A more successful workaround is introduced by [PR1189: Fix ROOT-8839](https://github.com/root-project/root/pull/1189). It patches LLVM using `find_package(ZLIB)`. This allows us to override the system zlib. A major disadvantage to this approach is that it only fixes the case for zlib. Moreover, the produced patch cannot be accepted in LLVM's mainline (based on private discussions with the code maintainers) introducing technical depth.

An optimal solution is to implement symbol resolution rules which follow the linker's where possible. Special, non-conforming to the linker, resolution will only happen for lookups in `clang` or `llvm` namespace. Such flexible implementation will solve the described issue in a generic way at effectively zero performance penalty.


## Detailed design

When LLVM IR is lowered to machine code the JIT is responsible for resolving all symbols. Some of them might be in locations such as shared libraries. The current resolution model goes through `llvm::DynamicLibrary::getAddressOfSymbol`. It iterates over the executable/shared library the JIT resides in. If the symbol name remains unresolved it iterates over the list of dynamically loaded library from earliest loaded onwards.

The proposed solution circumvents the symbol lookup in the executable first, allowing the loaded libraries to resolve the name first. Note that the way of iterating the libraries for symbol name resolution is significant but outside of the scope of the current proposal.

The challenge is to implement the symbol name lookup in a selective way. That is, whenever there is a symbol resolution request it should be resolved from external libraries first only if it not coming from namespace `llvm` or `clang`. There are two cases one needs to address during symbol resolution:
  * Direct -- the symbol contains `llvm` or `clang` in its mangled name. It is trivial.
  * Indirect -- the symbol does not contain the above information in its mangled name. Lets come back to our zlib example, if the symbol name is from zlib we will need context to decide how to resolve it. If the lookup is in the context of `llvm` or `clang` we should ask the executable first, otherwise we should ignore it and iterate the loaded libraries.

The implementation will enhance `cling::IncrementalJIT::getSymbolAddressWithoutMangling` by adding selective, context-based symbol name lookup. This will allow ROOT to override in a safe way the low-level support libraries which are linked against LLVM and thus in `libCling`.

## Source compatibility

No foreseen effect on the source compatibility unless if there are cases which rely on such erroneous behavior.

## Effect on ABI stability

No foreseen effect on the ABI stability is expected.

## Alternatives considered

No other alternatives, different to the explained workarouds, were considered. Authors are unaware of any other ways of resolving the issues but open to ideas.
