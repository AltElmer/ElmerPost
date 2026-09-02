# ElmerPost

ElmerPost is the original finite element post-processor for [Elmer FEM](https://github.com/ElmerCSC/elmerfem), written from 1995 by Juha Ruokolainen: around 100,000 lines of C with a Tcl/Tk interface over an OpenGL renderer. Paraview has long since displaced it, and Elmer removed it from the main tree on 19 December 2025.

**This is a fork of [ElmerCSC/ElmerPost](https://github.com/ElmerCSC/ElmerPost)**, the repository Peter Råback moved it to. It exists to work out what that repository needs, so the answers can go back upstream. Three things are different here.

## 1. The history is back

Upstream's repository has three commits. The code has **255**, from 31 May 2005 to 22 December 2025.

The missing 252 are not lost, they were simply not carried across: 203 Subversion revisions live in Elmer's pre-GitHub repository at [sourceforge.net/p/elmerfem](https://sourceforge.net/p/elmerfem/), and 49 more in ElmerCSC/elmerfem itself. Both were recovered and grafted, and **the joins are proven rather than assumed**: the last Subversion state and the 2014 GitHub import have the same tree hash, `4bcb5254667f73ea3c9505d09b3a7812973aa71d`.

Author names are Subversion handles mapped to `<handle>@users.sourceforge.net`. Guessing at real identities from a handle would put wrong names on other people's commits.

## 2. The code is at the root

Upstream keeps it under `post/`, deliberately, so an old build script can symlink to it as a one-to-one copy of the Elmer directory. In a repository whose only subject is ElmerPost, that is a level of indirection with nothing on the other side of it.

Worth noting while comparing the two: the move also changed the mode of **635 files** from `100644` to `100755`, making every C source, header, README and licence file executable. The blob hashes are identical so nothing else changed, and it looks like an artifact of copying the tree through a filesystem without permission bits. Not reproduced here.

## 3. MATC is a submodule

Upstream carries a copy of MATC's sources in `post/matc`. That copy is **twelve years behind**: 9,523 lines against 9,819, missing the OpenMP work, the locale fixes, the prototype cleanups and the `str.c` miscompilation workaround. 21 of its 22 source files differ from the maintained version.

It is now a submodule of [AltElmer/matc](https://github.com/AltElmer/matc), which is the same code with its history and its maintenance.

This is the argument for modularization stated as a fact rather than an opinion: **the vendored copy carried a bug that had already been fixed elsewhere.** MATC's `com_init()` takes a callback declared with an unspecified argument list, C23 redefines `()` as `(void)`, and every compiler defaulting to C23 rejects all 113 call sites. One copy of a file has that fixed. The other did not, because nobody knew it was the same file.

## Upstream does not build on its own, and here is why

Cloning ElmerCSC/ElmerPost and building it produces **113 errors**, every one of them in the vendored MATC and none in ElmerPost's own code. Four things are wrong, and they are all the same kind of thing: settings the Elmer tree supplied that an extracted repository does not inherit.

| what | where it came from |
| --- | --- |
| No C standard is set, so a C23 compiler rejects MATC's callbacks — and, once MATC is fixed, ElmerPost's own `initglp`, `gra_cylinder`, `gra_set_projection` and six `user_hook_*` callbacks for the same reason | Elmer's root `CMakeLists.txt`, **line 78**: `SET(CMAKE_C_STANDARD 99)` |
| `MINGW32` is set as a CMake cache variable, which the preprocessor never sees, so all 21 `#ifdef MINGW32` guards take the Unix branch and pull in `GL/glx.h` and X11 on Windows | Elmer's root `CMakeLists.txt`, **line 726**: `ADD_DEFINITIONS(-DMINGW32)` |
| `INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/../matc/src)` resolves to a directory that does not exist outside the Elmer tree | only ever correct as a sibling of `elmerfem/matc` |
| `Window` assigned from `HWND` in two places, which GCC 14 turned from a warning into an error | not inherited, just old |

A fifth is ordering rather than absence: the `IF(MINGW)` block sits after `ADD_SUBDIRECTORY`, and `ADD_DEFINITIONS` only reaches targets created after it, so putting the definition there would still not have worked.

## What works, and what does not

**Builds and links** on Linux with GCC, Clang and Intel `icx`/`ifx`, on x86_64 and arm64, and on Windows with MinGW-w64. Produces `ElmerPost` and `sico2elmer`.

**Starts** on Windows: it resolves `ELMER_POST_HOME`, loads `tcl/init.tcl` and brings up its Tcl interpreter.

**Does not open its graphics window on Windows.** GLAUX's `CreateWindow` fails with `ERROR_INVALID_HANDLE` and the program shows a dialog reading *"create window failed"*. To be clear about whose problem this is, it was checked against upstream's own build from unmodified sources: the identical dialog. It is not a regression introduced here, and the cause is **not established** — the obvious candidate, an ANSI/wide mismatch between `RegisterClass` and `CreateWindow`, was tested and refuted, since `UNICODE` is not defined for that file and forcing the explicit `RegisterClassA`/`CreateWindowA` changes nothing.

Linux is the platform whose GLX path was the one actually in use, so there is a CI job that runs ElmerPost under Xvfb and reports what it sees. It reports rather than enforces: the point is to find out whether this is Windows-only, and a job that failed the build would say less than one that prints the window tree.

## Building

```
git clone --recurse-submodules https://github.com/AltElmer/ElmerPost
cd ElmerPost
cmake -S . -B build
cmake --build build
```

Needs CMake 3.10, C, C++ and Fortran compilers, OpenGL, GLU and Tcl/Tk. On Debian and Ubuntu: `gfortran libgl1-mesa-dev libglu1-mesa-dev tcl-dev tk-dev libx11-dev libxext-dev libxi-dev libxmu-dev`. Under MSYS2 UCRT64: `mingw-w64-ucrt-x86_64-{gcc,gcc-fortran,tcl,tk}`.

## Licence

GPL 2.0, following the Elmer repository this came from. MATC is LGPL and is a submodule with its own licence. The bundled `tcl_license.terms` and `tk_license.terms` cover the Tcl/Tk code included here.

## Related

Part of an effort to make Elmer's components separable, discussed upstream in [ElmerCSC/elmerfem#202](https://github.com/ElmerCSC/elmerfem/issues/202). Siblings: [AltElmer/eio](https://github.com/AltElmer/eio), [AltElmer/matc](https://github.com/AltElmer/matc), [AltElmer/meshgen2d](https://github.com/AltElmer/meshgen2d), [AltElmer/elmerfront](https://github.com/AltElmer/elmerfront).

This is not an official CSC distribution.
