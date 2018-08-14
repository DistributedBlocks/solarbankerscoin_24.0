
# Skycoin C client library

[![Build Status](https://travis-ci.org/skycoin/skycoin.svg)](https://travis-ci.org/skycoin/skycoin)
[![GoDoc](https://godoc.org/github.com/skycoin/skycoin?status.svg)](https://godoc.org/github.com/skycoin/skycoin)
[![Go Report Card](https://goreportcard.com/badge/github.com/skycoin/skycoin)](https://goreportcard.com/report/github.com/skycoin/skycoin)

Skycoin C client library (a.k.a libskycoin) provides access to Skycoin Core
internal and API functions for implementing third-party applications.

## API Interface

The API interface is defined in the [libskycoin header file](/include/libskycoin.h).

## Building

```sh
$ make build-libc
```

This command compiles wrappers partially generated by [cgogen](https://github.com/simelo/cgogen) tool.

**Important** : `libskycoin.so` shared library
[can not be compiled on Windows 64 bit systems](t.me/skycoindev/925) due to
[lack of support for c-shared buildmode in golang 1.9](https://github.com/golang/go/issues/23582).
Make sure you have installed a go version higher than `1.10`.

## Testing

In order to test the C client libraries follow these steps

- Install [Criterion](https://github.com/Snaipe/Criterion)
  * locally by executing `make instal-deps-libc` command
  * or by [installing Criterion system-wide](https://github.com/Snaipe/Criterion#packages)
- Run `make test-libc` command

## Binary distribution

The following files will be generated

- `include/libskycoin.h` - Platform-specific header file for including libskycoin symbols in your app code
- `build/libskycoin.a` - Static library.
- `build/libskycoin.so` - Shared library object.

In Mac OS X the linker will need extra `-framework CoreFoundation -framework Security`
options.

In GNU/Linux distributions it will be necessary to load symbols in `pthread`
library e.g. by supplying extra `-lpthread` to the linker toolchain.


## API usage

The C API (a.k.a libskycoin) exposes the internals of Skycoin core
classes and objects. This makes it suitable for writing third-party
applications and integrations. The notable differences between go lang
and C languages have consequences for the consumers of the API.

### Data types

Skycoin core objects may not be passed across API boundaries. Therefore
equivalent C types are defined for each Skycoin core struct that
might be needed by developers. The result of this translation is
available in [skytpes.h](../../include/skytypes.h).

### Memory management

Caller is responsible for allocating memory for objects meant to be
created by libskycoin API. Different approaches are chosen to avoid
segmentation faults and memory corruption.

API functions perform memory allocation for output `GoString *` arguments.
In that case new memory is allocated dynamically by `libskycoin` code.
The caller C code is responsible for releasing that memory by passing the pointer
in `p` field in to [free()](http://en.cppreference.com/w/c/memory/free).

The parameters corresponding to slices returned by `libskycoin` are
of `GoSlice *` type. Aforementioned approach is also implemented for slices
with `cap` field set to `0` prior to function invocation.
Otherwise their `data` field must always be
set consistently to point at the buffer memory address whereas
`cap` must always be set to the number of items of the
target element type that will fit in the memory
area reserved in advance for that buffer. If the size of the data
to be returned by a given libskycoin function exceeds the value
set in `cap` then libskycoin will copy `cap` items in available
memory space and will set `len` to a negative value representing
the number of extra items that could not be copied due to
overflow.

For instance if `100` bytes have been allocated in advance
by the caller for a struct type that occupies `25` bytes then only
`4` items fit in that memory and `cap` should be set accordingly.
In the hypothetical situation that `libskycoin` result occupies
`125` bytes (e.g. a slice of same type including `5` items) then
the first `100` bytes will be copied onto C-allocated `data` buffer
and `len` field will be set to `-1` as a side-effect of function
invocation. The caller will be responsible for
[reallocating another memory buffer](http://en.cppreference.com/w/c/memory/realloc)
using a higher `cap` and retry.
