-----
title: Haskell and Linking with C++
author: Arjun Comar
description: A story about Haskell interop with C++
-----

I should probably open this with a caveat that Haskell's interop story with C++ isn't quite as bad as it's 
made out to be. Yes, GHC can't compile and link directly against a C++ library, but there aren't many languages
that can. Once C wrappers are present for the library, it's a relatively straight forward process to make the
library available in Haskell.

But it's not painless. The build tools assume that you're working with a C library and that you want to use C
buildtools. So you have to trick Cabal and GHC into using `g++` instead of `gcc`. Ok, no big deal. Except that
it doesn't take long before you run up against edge cases and the law of leaky abstractions.

I've discovered these issues while working on my [OpenCV wrappers](https://github.com/arjuncomar/opencv_contrib) 
and my slow work on an [idiomatic Haskell library](https://github.com/arjuncomar/revelation) that builds on those 
wrappers. The wrappers more or less work but the library is progressing much more slowly than I'd like. The reasons
for this are unrelated to the problems outlined in this post but they sure don't help!

This is a long post detailing my saga wrestling with cabal, ghc, g++, and ld. I'll be updating it as I discover new
issues even after this goes live. Hopefully someone else finds this useful. If you'd like any extra information
on any problems I bring up or manage to resolve, ask, and I'll add the relevant details to this post.

OpenCV Wrappers
---------------

A little background on how the wrappers work first so that the rest of this story makes more sense. The wrappers live
in the opencv_contrib repo (pull request pending for official inclusion) and are built along with the full OpenCV
source tree. Instructions for how to do this are on the 
[opencv_contrib repo page](https://github.com/arjuncomar/opencv_contrib).

The wrappers are generated as separate C and Haskell modules for the OpenCV tree via CMake. This was in and of itself
relatively painful to set up until I decided to scrap attempting to actually compile the Haskell bindings with CMake
and had CMake call out to Cabal instead. There's a small linking issue here, but that's easily solved by passing g++ as
the linker GHC should use. In any case, short python scripts use a header parser written for the python bindings
to produce the bindings and this is done via CMake against the full available OpenCV source tree. Several classes
(the templated ones) are missed by the header parser, and so these modules are wrapped by hand.

Linking Issues
--------------

As I mentioned previously, the first linking issue comes within the build process for the raw Haskell bindings. It
effectively boils down to C++ libraries (like the STL) being required but `gcc` doesn't know where to find them. Asking
`g++` to be both the 'C' compiler and the linker solves this issue just fine. CMake is therefore set to call `cabal` 
with `--with-gcc={CMAKE_CXX_COMPILER}`. And the cabal file asks `ghc` to use `g++` as the linker via `-pgml g++`.

But it gets worse. When I was initially developing this project, everything was thrown together into one project --
the Haskell library, the C and Haskell bindings, and the tiny 
[C++ interop library](https://github.com/arjuncomar/cpp-interop) that I'm adding to occasionally. In this state,
linking was not problematic and everything built without any issues. As I split these libraries apart though,
I started running into bigger and bigger issues.

The first of these was that GHC couldn't link an executable against the built C module that housed the wrappers.
Every symbol referenced from the `.so` file would throw a linker error when the symbol couldn't be located. I spent
some time verifying that the symbols referenced were indeed contained within the module via `nm`.

I took the issue to #haskell and one of the first questions I was asked was if I was trying to pass a static lib,
in which case the argument order to the linker would matter, and Cabal would fail to build the executable with the
issues I was facing. I wasn't, but the question gave me an idea nonetheless. I switched things around in CMake to have
it construct a `.a` file and began to debug from there. At first, I was having issues because I couldn't get cabal to 
pass the `.a` file as the last (or late) argument to the linker.

The problem, it turns out, is that when passed a static library, ld only keeps the symbols that the archive provides
that it already knows are not referenced. If further on the command line another library requires symbols provided by
the static archive, you're SOL. This behavior can be switched off by passing the --whole-archive option to the linker
just before passing the static archive. Of course, passing this option all the way through to ld isn't exactly easy,
and the solution I constructed was to stick the following in the cabal file for the executable:

    ghc-options: -pgml g++ "-optl-Wl,--whole-archive" "-optl-Wl,-Bstatic" "-optl-Wl,-lopencv_c" "-optl-Wl,-Bdynamic" "-optl-Wl,--no-whole-archive"

`-pgml g++` is required once again because there are references to the C++ stdlib and other libs that `g++` knows how to
find but `gcc` doesn't. The rest is a carefully crafted sequence of arguments to the system linker on which C module
archive to load (the static one) and to read the entire thing in and keep all of it. This increases the size of the
executable but it does allow the program to correctly compile and link. However, it still requires the dynamic `.so` libs
to be available on the `LD_LIBRARY_PATH` in order to actually run.

GHC 7.8
-------

Up to this point, `ghci` and consequently `cabal repl` fail to read in and load the necessary symbols. My understanding
was that the issue was more or less resolved in GHC 7.8 and so I waited for the release with anticipation. But to my
consternation, the issue is more or less the same. But in addition, `ghc-mod` has also stopped working with similar
errors. I'd also like to make use of Template Haskell to reduce a large swath of necessary boilerplate, but can't
because `ghci` fails to load. The issue is one I don't really have my head around yet, but I'd very much love input
and ideas on.

The `ghc-mod` issue is particularly puzzling because the program used to work just fine. I haven't yet investigated in
sufficient depth to determine if a bug report is necessary or if it's an error on my end. The program complains about
a C++ symbol being unknown:

    ghc-mod: /home/arjun/src/revelation/.cabal-sandbox/lib/x86_64-linux-ghc-7.8.2/cpp-interop-0.1.0.0/HScpp-interop-0.1.0.0.o: unknown symbol `_Znwm'
    Revelation/Mat.hs:0:0:Error:ghc-mod: unable to load package `cpp-interop-0.1.0.0'

That symbol is indeed listed within the archive as unknown, but if the C++ linker were being employed, it and the other
missing symbols would be easily found. The code compiles and runs after all. *And* that particular library can be
loaded into `ghci` without any issues.

`ghci` on the other hand seems unable to find symbols that are undefined in one archive but are specified in another
archive provided on its command line. For example, trying to load the revelation library into the repl yields the
following:

    <command line>: user specified .o/.so/.DLL could not be loaded (/usr/local/lib/x86_64-linux-ghc-7.8.2/opencv-3.0.0/libHSopencv-3.0.0-ghc7.8.2.so: undefined symbol: cv_create_BFMatcher)

But nm shows the symbol is very much present in the text section of `libopencv_c.so`. Even providing both archives
(along with all other required libaries) directly on the command line to ghci doesn't resolve the issue and it fails 
the same way.

Other Issues
------------

As recently as today (4/16/2014) I've bumped into a new issue. When trying to link the final executable I now get

    /usr/bin/ld: dist/build/Test/Test: hidden symbol `cv_erode' in /usr/local/lib/libopencv_c.a(opencv_generated.cpp.o) is referenced by DSO
    /usr/bin/ld: final link failed: Bad value

Wonder what I broke... Oh, as I was messing with cabal options, I turned on some extra library compilation option
(I removed most of the ones I added in the last few days, I should go back and do this more scientifically) and
that broke my build. Whoops.

This build is entirely too fragile, but I'm not sure how to make it more robust. Hopefully
someone else in the Haskell community has run into these issues and knows how to get these tools to work together
more gracefully. I'm open to going as far as to scrap my entire build process and rewriting it from scratch.

Final Thoughts
--------------

I'm genuinely puzzled by a lot of the issues I'm running into. I think the problem starts from the fact that I don't
really understand how `cabal` and `ghc` actually build my projects, so my current plan is to improve my understanding
of this process. Any resources people have on either would be much appreciated. As are thoughts on this entirely
ludicrous build process I've kluged together.
