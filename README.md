[![Build Status](https://travis-ci.org/spearsem/buffersort.svg?branch=master)](https://travis-ci.org/spearsem/buffersort)

# buffersort
Provide a variety of sorting algorithms that operate in-place on types that implement the Python buffer protocol.

## install
`buffersort` requires Cython >= 0.22 and NumPy >= 1.10.1. `buffersort` builds are [tested with travis-ci](https://travis-ci.org/spearsem/buffersort) for Python 2.7.11 and Python 3.5.0 on 64-bit Ubuntu 14.04.

### basic
Clone the repo, navigate to the top-level directory where you can find `setup.py` and use

```bash
python setup.py install
```

If you'd like to preserve a copy of the files affected by installation so that you can manually uninstall later, add the `--record` option:

```bash
python setup.py install --record install-files.txt
```

### conda
If you'd like to install with `conda` you can use the `meta.yaml` specification that comes with the source code under the `info/` directory. The package is not yet hosted with Anaconda, so the installation retrieves the code from GitHub. 

To avoid downloading the code twice, you can instead merely download what you need for installation -- namely, `meta.yaml` and the relevant build script for your platform. Place these in a directory named `info` and the commands below will work just as if you had cloned the entire repo.

Before installing, the best practice is to create a new `conda` environment first, such as

```bash
conda create -n testBuffersort python=2.7 numpy>=1.10.1 cython>=0.22
```

Then use `source activate testBuffersort` to ensure that `conda` is installing `buffersort` into your test environment. If everything installs satisfactorally, you may delete the temporary environment and repeat installation into your working environment.

To install with `conda` navigate to the top-level of the downloaded source and build the package locally. Alternatively, if you elected to grab only `meta.yaml` and the build script, specify the path to the directory in which they reside.

```bash
conda build info
#... conda build /path/to/recipe-dir/
```

This will download the code and prepare a build within the internal package directories maintained by `conda`. It will not affect your local directory. 

After `conda build` finishes, the built packages must be installed to be available.

```bash
conda install --use-local buffersort
```

and `conda` will refer to the local build it maintains after the execution of `conda build` for the installation.

### pip
Pip instructions are forthcoming.


## usage
Assuming you have installed the package, everything can be accessed by importing `buffersort`:

```python
>>> import buffersort
>>> from array import array
>>> a = array('l', [-1, -50, 10, -3, 27, 14])
>>> a
array('l', [-1, -50, 10, -3, 27, 14])

>>> buffersort.heap_sort(a)
>>> a
array('l', [-50, -3, -1, 10, 14, 27])
```

## testing
`buffersort` provides unit tests as a built package so that you may execute them more conveniently. The test subpackage, `buffersort.test`, may be imported directly:

```python
>>> import buffersort.test as test
```

Alternatively, the `buffersort.test.test_buffersort` module, which contains the unit tests, is provided as part of the top-level `buffersort` package import, and you may run the full suite of unit tests as follows.

```python
>>> import buffersort
>>> buffersort.test_buffersort.run_tests()
```

If tests fail due to type signature errors or unsupported type errors, feel free to create an issue with your platform information. The type support is controlled by the `Ord` fused type found in `buffersort.pxd`. Cython states that support for fused types is experimental, so some bugs may be unavoidable due to Cython fused type support. Other bugs may be fixable by extending the library to include additional types in the `Ord` fused type definition. 

## motivation
`buffersort` makes use of Cython fused types and typed memoryviews to enable writing concise functions in simple Cython syntax from which many overloaded versions are automatically generated and correctly dispatched in the compiled code. 

`buffersort` is meant to be an educational library for the task of writing Python modules that include functions which are both *performant* and *polymorphic*. Traditionally, with Python's dynamic type model, there is often a trade-off between writing a generic function accepting of many different Python objects and writing specialized functions that are tightly compiled and optimized to handle a narrow type. But with Cython fused types, it is possible to make functions more generic without sacrificing any of their static type specialization. 

The use of `Ord` as a Cython fused type is not accidental. This motif is inspired by the [`Ord` type class in the Haskell programming language](https://hackage.haskell.org/package/base-4.8.2.0/docs/Data-Ord.html). Part of the goal of `buffersort` is to demonstrate how, at least for a limited set of situations, Cython fused types enable designs that are similarly easy to work with and expressive as Haskell type classes.

For example, we can see from a small section of the Cython source code how a C-level function is defined with static type information, yet corresponds to autogenerated functions that cover all necessary static type signatures from all constituent types of `Ord`:

```cython
cdef void _swap(Ord[:] buf, int i, int j):
    """
    Swap elements i and j in buffer buf.
    """
    cdef Ord temp = buf[i]
    buf[i] = buf[j]
    buf[j] = temp
```

In this case, `_swap` is a simple helper function to swap the elements residing at positions `i` and `j` of an array. The array argument `buf` has a static type of `Ord[:]` -- a typed memoryview of a one-dimensional array of `Ord` values. 

In the body of the function, we can work with the array without caring whether it is an array of `int` or an array of `double` -- the overloaded versions specific to these types will be autogenerated since those base types are part of the `Ord` fused type. We can even create a temporary value, `temp` with type `Ord` that will be correctly resolved for each distinct compiled overload of the function.