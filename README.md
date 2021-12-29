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
`import_utils.cmake` was created to solve The CMake Windows Library Problem™,
an issue I've only recently become aware of due to CMake's evolving dependency-management support.

Basically, the majority of Windows `IMPORTED` library targets created by Find modules are broken.
As a result, they're not compatible with CMake's runtime-dependency discovery features.
Tools like `$<TARGET_RUNTIME_DLLS>` work fine with the `EXPORTED` targets CMake creates,
but the `IMPORTED` targets created by Find modules are nearly always configured incorrectly.
Use `import_utils` to fix those targets (no matter where they were created),
so that all of your `CONFIG`- and `MODULE`-mode dependencies will be handled properly.

For a more detailed explanation of the issue itself, see the [Background](#Background) section below.

## Usage

This module is intended for use by Find module authors,
to aid them in creating correct targets on Windows platforms.
I had hoped that it would be useful to endusers as well,
so they could repair bad targets produced by Find modules.
Unfortunately, see [issue #1](../../issues/1) for why that doesn't work.

When writing a Find module, just copy `import_utils.cmake` into your project,
`include()` it, and create `SHARED IMPORTED` targets for your libraries,
even on Windows. **No more `UNKNOWN IMPORTED` targets** — they break things.

To build library targets, continue using `find_library()` same as before,
and use the import library path (`.dll.a` / `.lib`) it returns as normal.
Continue placing it into the target's `IMPORTED_LOCATION`,
even though that's the wrong property.
Call <code>fix_imports(TARGET <var>name</var>)</code> afterwards,
and the paths will be updated for a valid `SHARED IMPORTED` target.

### A simplified example `FindDependency.cmake` module

```cmake
include(${CMAKE_CURRENT_LIST_DIR}/import_utils.cmake)

find_library(Dependency_LIBRARY
  NAMES dependency
  PATHS
    ENV DEPENDENCY_DIR
)
find_library(Dependency_DEBUG_LIBRARY
  NAMES dependency-d dependency_debug
  PATHS
    ENV DEPENDENCY_DIR
)

find_path(Dependency_INCLUDE_DIR
  dependency.h
  PATHS
    ENV DEPENDENCY_DIR
  PATH_SUFFIXES
    include
    include/dependency
)

if (Dependency_LIBRARY)
  list(APPEND Dependency_LIBRARIES ${Dependency_LIBRARY})
endif()
if (Dependency_DEBUG_LIBRARY)
  list(APPEND Dependency_LIBRARIES ${Dependency_DEBUG_LIBRARY})
endif()
if (Dependency_INCLUDE_DIR)
  list(APPEND Dependency_INCLUDE_DIRS ${Dependency_INCLUDE_DIR})
endif()

include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(Dependency
  REQUIRED_VARS
    Dependency_LIBRARIES
    Dependency_INCLUDE_DIRS
)

if (NOT TARGET Dependency::lib)
  add_library(Dependency::lib SHARED IMPORTED)
  
  set_property(TARGET Dependency::lib PROPERTY
    IMPORTED_LOCATION ${Dependency_LIBRARY})
    
  if (Dependency_DEBUG_LIBRARY)
    set_property(TARGET Dependency::lib PROPERTY
      IMPORTED_LOCATION_DEBUG ${Dependency_LIBRARY})
    set_property(TARGET Dependency::lib APPEND PROPERTY
      IMPORTED_CONFIGURATIONS DEBUG)
  endif()
  
  set_property(TARGET Dependency::lib PROPERTY
    INTERFACE_INCLUDE_DIRECTORIES ${Dependency_INCLUDE_DIRS})
    
  # This will correct IMPORTED_LOCATION, as well as any
  # other configurations set in IMPORTED_CONFIGURATIONS
  fix_imports(TARGET Dependency::lib)
  
endif()
```
That's all there is to it.
The target will be created properly regardless of platform.

## Background

**Q:** Why is this necessary? Why are so many Find modules broken?

**A:** Because the way `find_library()` works on Windows platforms
is different from how it works on every _other_ platform.

There are two problems with how Find modules work on Windows,
and they make it incredibly difficult to create `SHARED IMPORTED` targets.

1. A Windows `SHARED` library target needs to have _two_ properties set, not just one:
   * `IMPORTED_LOCATION` — the location of the actual runtime `.dll` file
   * `IMPORTED_IMPLIB` — the DLL's import library, usually named `.lib` or `.dll.a`

2. CMake's `find_library()` **doesn't** find _library_ files (DLLs) on Windows,
   it finds _import library_ files.
   So, what `${Dependency_LIBRARY}` actually holds, on Windows,
   is the path to a `dependency.dll.a` file that's only useful to the linker.
   Then we went and put that into the **wrong property** (`IMPORTED_LOCATION`).
   The `.dll.a` (or `.lib`) file's path belongs in `IMPORTED_IMPLIB`.

   Meanwhile, we have no idea where the `dependency.dll` file is,
   even though _that's_ what we're supposed to put into `IMPORTED_LOCATION`.
   And CMake needs to have that path to do runtime library management.

Running `fix_imports()` on a target addresses both of those problems.
It takes the import library path from `IMPORTED_LOCATION`,
uses it to find the `.dll` file, and updates `IMPORTED_LOCATION`.
(It also moves the import library path to `IMPORTED_IMPLIB`.)

So we can now create `SHARED IMPORTED` targets even on Windows.
`fix_imports()` will find the library's DLL file and fix the paths.
And because it _expects_ the import library path to be in `IMPORTED_LOCATION`,
you can keep sharing the same code between Windows and non-Windows systems.

`fix_imports()` does absolutely nothing on non-Windows platforms,
and it ignores any targets that are already configured correctly.
So it's generally safe to leave it in your code unconditionally.

Even if you're not sure what OS the build will be running on,
if Windows might be one of them, add it in.
You could do an `if (WIN32)` check first,
but it's not really necessary since there's an implicit one in `import_utils`.

Multiple-configuration targets are supported.
`fix_imports()` processes `IMPORTED_LOCATION_*` for every configuration
listed in the target's `IMPORTED_CONFIGURATIONS`.

If necessary, you can add some optional parameters to `fix_imports()` calls:

* <code>fix_imports(TARGET <var>name</var> PACKAGE_NAME <var>prefix</var>)</code>

  The `find_program()` call used to locate a library's DLL
  will be passed <code><var>prefix</var>_RUNTIME_DLL</code>
  (or `_RUNTIME_DLL_<config>`)
  as the name of the cache variable to hold the results.
  If you don't pass in a name, it's generated from the target name.
 
* <code>fix_imports(TARGET <var>name</var> PATHS <var>path1 ...</var>)</code>

  By default, DLL files will be searched for in these locations:
  
  * The directory where the import library is located
  * A directory `bin/` in the same location as its parent
    (so if an import library is located in the directory
    `.../path/lib/implib.dll.a`, the DLL will be looked for in
    `.../path/bin/`)
  * The directories in the `PATH` environment variable.
  * Any other default locations CMake searches automatically

  If necessary, use the `PATHS` option to add to the list.

## License

The code is licensed under the
[Creative Commons CC0-1.0 license](https://creativecommons.org/publicdomain/zero/1.0/legalcode),
which basically means you can do whatever you want with it.
Modify it, redistribute it, sell it,
remove my name and pass it off as your own — anything goes.
I really don't care.
