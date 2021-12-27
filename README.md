<!--
© 2021 FeRD (Frank Dana)

SPDX-License-Identifier: CC0-1.0
-->


# CMake Win32 IMPORTED library utils

Tools for managing/correcting CMake `IMPORTED_LOCATION` and `IMPORTED_IMPLIB` properties on Windows library targets.

## Motivation

This tool began life with
[this post](https://discourse.cmake.org/t/windows-libraries-find-modules-and-target-runtime-dlls-re-re-revisited/4286)
I made to the CMake Discourse community forum.
`implib_utils.cmake` was created to solve The CMake Windows Library Problem™,
an issue I've only recently become aware of due to CMake's evolving dependency-management support.

Basically, the majority of Windows `IMPORTED` library targets created by Find modules are broken.
As a result, they're not compatible with CMake's runtime-dependency discovery features.
Tools like `$<TARGET_RUNTIME_DLLS>` work fine with the `EXPORTED` targets CMake creates,
but the `IMPORTED` targets created by Find modules are nearly always configured incorrectly.
Use `implib_utils` to fix those targets (no matter where they were created),
so that all of your `CONFIG`- and `MODULE`-mode dependencies will be handled properly.

For a more detailed explanation of the issue itself, see the [Background](#Background) section below.

## Usage

There's very little to it: the tool is one file, one function.
Just copy `implib_utils.cmake` into your project, `include()` it,
and repair any broken library targets by calling `fix_implib()` on them.

`implib_utils.cmake` is a single, self-contained CMake module file.
It provides one user-facing function, `fix_implib()`,
which is useful both when writing CMake Find modules,
and when maintaining CMake projects that use `find_package()` on Windows.

Simply copy it into your project directory and include the module:

```bash
cd /my/project/repo
cp implib_utils.cmake ./cmake/
```

```cmake
list(APPEND CMAKE_MODULE_PATH cmake)
include(implib_utils)

# Use FindZLIB.cmake to create a broken UNKNOWN IMPORTED target
find_package(ZLIB REQUIRED)

# Fix the target's IMPORTED_LOCATION and IMPORTED_IMPLIB
fix_implib(TARGET ZLIB::ZLIB)

target_link_libraries(my_exe PUBLIC ZLIB::ZLIB)
```

## Details

What's the problem with CMake Find modules on Windows?
Why are so many Find modules broken?

Because the way `find_library()` works on Windows platforms
is different from how it works on every _other_ platform.

A Find module typically does something like this (simplified):
```cmake
find_library(DEP_LIBRARY dependency)
find_path(DEP_INCLUDE_DIRECTORY dep.h)

add_library(DEP::DEP SHARED IMPORTED)
set_target_properties(DEP::DEP PROPERTIES
  IMPORTED_LOCATION ${DEP_LIBRARY}
  INTERFACE_INCLUDE_DIRECTORIES ${DEP_INCLUDE_DIRECTORY}
)
```
And that's it, you've got a shared library target.
But that code not won't work on Windows due to the `SHARED IMPORTED` target type,
at which point most of us generally just change it to `UNKNOWN IMPORTED`.
That works, so we call it a day.

There are two problems with that, and they come down to the reason `SHARED IMPORTED` fails.

1. A Windows `SHARED` library target needs to have _two_ properties set, not just one:
   * `IMPORTED_LOCATION` — the location of the actual runtime `.dll` file
   * `IMPORTED_IMPLIB` — the DLL's import library, usually named `.lib` or `.dll.a`

2. CMake's `find_library()` actually **doesn't** find library files (DLLs) on Windows,
   it finds _import library_ files.
   So, `${DEP_LIBRARY}` is actually the path to a `libdep.dll.a` file,
   which we put into the **wrong property** (`IMPORTED_LOCATION`).
   That path belongs in `IMPORTED_IMPLIB`.
   Meanwhile, we have no idea where the `libdep.dll` file is,
   even though _that's_ what we're supposed to put into `IMPORTED_LOCATION`.

Running `fix_implib()` on a target addresses both of those problems.
It takes the import library path from `IMPORTED_LOCATION`,
uses it to find the `.dll` file, and updates `IMPORTED_LOCATION`.
(It also moves the import library path to `IMPORTED_IMPLIB`.)

If you're a Find module author,
you can now create `SHARED IMPORTED` targets even on Windows,
because `fix_implib()` will find your library's DLL file and fix the target properties.
And because it _expects_ the import library path to be in `IMPORTED_LOCATION`,
you can continue to share the same code between Windows and non-Windows systems.

If you're writing a `CMakeLists.txt` file,
you should run `fix_implib()` after calling `find_package()`,
if the package's `IMPORTED` targets were created by a Find module.

(In an ideal world, Find modules would all use `implib_utils` to create good targets,
so users writing `CMakeLists.txt` files don't have to worry about any of this.
But we're still a long way off from there.)

`fix_implib()` does absolutely nothing on non-Windows platforms,
and it ignores any targets that are already configured correctly.
So, it's generally safe to leave it in your code unconditionally,
even if you're not sure what OS the build will be running on,
or whether the `IMPORTED` targets produced by a `find_package()` call
have been created by a Find module or a `Config.cmake` package.

Multiple-configuration targets are also supported.
`fix_implib()` processes `IMPORTED_LOCATION_RELEASE`,
`IMPORTED_LOCATION_DEBUG`, etc. in addition to `IMPORTED_LOCATION`.

## License

The code is licensed under the
[Creative Commons CC0-1.0 license](https://creativecommons.org/publicdomain/zero/1.0/legalcode),
which basically means you can do whatever you want with it.
Modify it, redistribute it, sell it,
remove my name and pass it off as your own — anything goes.
I really don't care.
