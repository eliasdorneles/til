---
title: "About Linux and library loading"
date: "2025-05-19T20:16:30+02:00"
tags: []
---
The Linux documentation project has a nice [Program Library
HOWTO](https://tldp.org/HOWTO/Program-Library-HOWTO/) explaining several
concepts around libraries on Linux.

These are some notes I took while reading through it, mostly to gain a better
understanding about shared libraries.

A [**static library**](https://tldp.org/HOWTO/Program-Library-HOWTO/static-libraries.html) self-contained archive containing object files,
has the `.a` suffix. When compiling a program and linking against it,
all code referenced by the program will be included in the final executable.

This will make a bigger binary, with an impossible to update dependency, but
it's also handy because no dependencies.

[**Shared
libraries**](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
are libraries which are loaded automatically by programs when they start. They
use specific binary formats, and there are several rules and options and how to
use them.

[**Dynamic libraries
(DLLs)**](https://tldp.org/HOWTO/Program-Library-HOWTO/dl-libraries.html)
are essentially "plug-ins": libraries that loaded programmatically by the
code via a specifig API. Share the same format as shared libraries, the only
difference is how they're used.

## More details about shared libraries

Shared libraries have a _soname_ and a _real name_.

The _soname_ is in a format like `lib${NAME}.so.${NUMBER}`

The _real name_ adds version and optionally release number at the end, like so: `lib${NAME}.so.${NUMBER}.{VERSION}.{RELEASE}`

### Example:

* soname: `libjack.so.0`
* fully qualified soname: `/usr/lib/x86_64-linux-gnu/libjack.so.0` (symlinks to the real name file)
* real name: `/usr/lib/x86_64-linux-gnu/libjack.so.0.1.0`  (actual ELF file)
* linker name: `libjack.so` (symlink to soname)

### Useful programs and config files:

`ldconfig`:
:   a program that helps to setup all those library links when installing a library

`ldd`:
:   a program that displays which shared libraries are used by a given binary

`ld`:
:   the linker, last step when compiling a program.
    it combines a bunch of object files into a single file w/ references resolved,
    which can be either an executable, or a shared library, or a static library,
    in different formats.

`/lib/ld-linux.so.X`:
:   The loader, Linux will run this automatically when running an ELF binary,
    It will find the shared libraries from the correct places and load them.
    Uses cache file /etc/ld.so.cache, which is kept up-to-date by ldconfig
    whenever a library is installed.

`/etc/ld.so.conf`:
:   configures behavior of the loader

`/etc/ld.so.preload`:
:   Any libraries here will be preloaded and take precedence over the standard set.
    This can be used for emergency patches.

### Environment variables

`LD_LIBRARY_PATH`
:   Allows overriding standard set of libraries, the loader will search here
    first if it's set.
    Handy for development and testing, but not used for installation.

    Alternatively, one can do:
    `/lib/ld-linux.so.2 --library-path $LIBRARY_PATH /path/to/executable`

`LD_PRELOAD`
:   Like `LD_LIBRARY_PATH`, but will also preload the libraries here.
    Basically, same as `/etc/ld.so.preload`, except works in user-level

`LD_DEBUG`
:   Used to debug the loader behavior.
    - `LD_DEBUG=files $COMMAND` will show processing of files and libraries, what is
    loaded and in what order.
    - `LD_DEBUG=bindings $COMMAND` will show information about symbols binding
    - `LD_DEBUG=libs $COMMAND` will show library search paths
    - `LD_DEBUG=versions $COMMAND` will show dependencies and versions
