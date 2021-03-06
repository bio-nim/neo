# Neo - A Matrix library

![logo](https://raw.githubusercontent.com/unicredit/neo/master/img/neo.png)

This library is meant to provide basic linear algebra operations for Nim
applications. The ambition would be to become a stable basis on which to
develop a scientific ecosystem for Nim, much like Numpy does for Python.

The library has been tested on Ubuntu Linux 16.04 64-bit using
either ATLAS, OpenBlas or Intel MKL. It was also tested on OSX Yosemite. The
GPU support has been tested using NVIDIA CUDA 8.0.

The library is currently aligned with latest Nim devel.

API documentation is [here](https://cdn.rawgit.com/unicredit/neo/master/htmldocs/neo.html)

A lot of examples are available in the tests.

Table of contents
-----------------
<!-- TOC depthFrom:2 depthTo:6 orderedList:false updateOnSave:true withLinks:true -->

- [Working on the CPU](#working-on-the-cpu)
  - [Dense linear algebra](#dense-linear-algebra)
    - [Introduction](#introduction)
    - [Initialization](#initialization)
    - [Working with 32-bit](#working-with-32-bit)
    - [Accessors](#accessors)
    - [Slicing](#slicing)
    - [Iterators](#iterators)
    - [Equality](#equality)
    - [Pretty-print](#pretty-print)
    - [Reshape operations](#reshape-operations)
    - [BLAS Operations](#blas-operations)
    - [Universal functions](#universal-functions)
    - [Rewrite rules](#rewrite-rules)
    - [Solving linear systems](#solving-linear-systems)
    - [Computing eigenvalues and eigenvectors](#computing-eigenvalues-and-eigenvectors)
    - [Linking BLAS and LAPACK implementations](#linking-blas-and-lapack-implementations)
  - [Sparse linear algebra](#sparse-linear-algebra)
- [Working on the GPU](#working-on-the-gpu)
  - [Linking CUDA](#linking-cuda)
  - [Dense linear algebra](#dense-linear-algebra-1)
  - [Sparse linear algebra](#sparse-linear-algebra-1)
- [TODO](#todo)
- [Contributing](#contributing)

<!-- /TOC -->

## Working on the CPU

### Dense linear algebra

#### Introduction

The library revolves around operations on vectors and matrices of floating
point numbers. It allows to compute operations either on the CPU or on the
GPU offering identical APIs.

The library defines types `Matrix[A]` and `Vector[A]`, where `A` is sometimes
restricted to be `float32` or `float64` (usually to use BLAS and LAPACK
routines). Actually, `Vector[A]` is just a small wrapper around `seq[A]`, which
allows to perform linear algebra operations on standard Nim sequences without
copying.

#### Initialization

Here we show a few ways to create matrices and vectors. All matrices methods
accept a parameter to define whether to store the matrix in row-major (that is,
data are laid out in memory row by row) or column-major order (that is, data
are laid out in memory column by column). The default is in each case
column-major.

Whenever possible, we try to deduce whether to use 32 or 64 bits by appropriate
parameters. When this is not possible, there is an optional parameter `float32`
that can be passed to specify the precision (the default is 64 bit).

Static matrices and vectors can be created like this:

```nim
import neo

let
  v1 = makeVector(5, proc(i: int): float64 = (i * i).float64)
  v2 = randomVector(7, max = 3.0) # max is optional, default 1
  v3 = constantVector(5, 3.5)
  v4 = zeros(8)
  v5 = ones(9)
  v6 = vector(1.0, 2.0, 3.0, 4.0, 5.0) # `vector` also accepts a seq
  m1 = makeMatrix(6, 3, proc(i, j: int): float64 = (i + j).float64)
  m2 = randomMatrix(2, 8, max = 1.6) # max is optional, default 1
  m3 = constantMatrix(3, 5, 1.8, order = rowMajor) # order is optional, default colMajor
  m4 = ones(3, 6)
  m5 = zeros(5, 2)
  m6 = eye(7)
  m7 = matrix(@[
    @[1.2, 3.5, 4.3],
    @[1.1, 4.2, 1.7]
  ])
```

All constructors that take as input an existing array or seq perform a copy of
the data for memory safety.

#### Working with 32-bit

Some constructors (such as `zeros`) allow a type specifier if one wants to
create a 32-bit vector or matrix. The following example all return 32-bit
vectors and matrices

```nim
import neo

let
  v1 = makeVector(5, proc(i: int): float32 = (i * i).float32)
  v2 = randomVector(7, max = 3'f32) # max is no longer optional, to distinguish 32/64 bit
  v3 = constantVector(5, 3.5'f32)
  v4 = zeros(8, float32)
  v5 = ones(9, float32)
  v6 = vector(@[1'f32, 2'f32, 3'f32, 4'f32, 5'f32]) # this `seq` shares data with the vector
  m1 = makeMatrix(6, 3, proc(i, j: int): float32 = (i + j).float32)
  m2 = randomMatrix(2, 8, max = 1.6'f32)
  m3 = constantMatrix(3, 5, 1.8'f32, order = rowMajor) # order is optional, default colMajor
  m4 = ones(3, 6, float32)
  m5 = zeros(5, 2, float32)
  m6 = eye(7, float32)
  m7: Matrix32[2, 3] = matrix(@[
    @[1.2'f32, 3.5'f32, 4.3'f32],
    @[1.1'f32, 4.2'f32, 1.7'f32]
  ])
```

One can convert precision with `to32` or `to64`:

```nim
let
  v64 = randomVector(10)
  v32 = v64.to32()
  m32 = randomMatrix(3, 8, max = 1'f32)
  m64 = m32.to64()
```

Once vectors and matrices are created, everything is inferred, so there are no
differences in working with 32-bit or 64-bit. All examples that follow are for
64-bit, but they would work as well for 32-bit.

#### Accessors

Vectors can be accessed as expected:

```nim
var v = randomVector(6)
v[4] = 1.2
echo v[3]
```

Same for matrices, where `m[i, j]` denotes the item on row `i` and column `j`,
regardless of the matrix order:

```nim
var m = randomMatrix(3, 7)
m[1, 3] = 0.8
echo m[2, 2]
```

One can also map vectors and matrices via a proc:

```nim
let
  v1 = v.map(proc(x: float64): float64 = 2 - 3 * x)
  m1 = m.map(proc(x: float64): float64 = 1 / x)
```

#### Slicing

The `row` and `column` procs will return vectors that share memory with their
parent matrix:

```nim
let
  m = randomMatrix(10, 10)
  r2 = m.row(2)
  c5 = m.column(5)
```

Similarly, one can slice a matrix with the familiar notation:

```nim
let
  m = randomMatrix(10, 10)
  m1 = m[2 .. 4, 3 .. 8]
  m2 = m[All, 1 .. 5]
```

where `All` is a placeholder that denotes that no slicing occurs on that
dimension.

In general it is convenient to have slicing, rows and columns that do not
copy data but share the underlying data sequence. This can have two possible
drawbacks:

* the result may need to be modified while the original matrix stays unchanged,
  or viceversa;
* a small matrix or vector may hold a reference to a large data sequence,
  preventing it to be garbage collected.

In this case, it is enough to call the `.clone()` proc to obtain a copy
of the matrix or vector with its own storage.

#### Iterators

One can iterate over vector or matrix elements, as well as over rows and columns

```nim
let
  v = randomVector(6)
  m = randomMatrix(3, 5)
for x in v: echo x
for i, x in v: echo i, x
for x in m: echo x
for t, x in m:
  let (i, j) = t
  echo i, j, x
for row in m.rows:
  echo row[0]
for column in m.columns:
  echo column[1]
```

One important point about performance. When iterating over rows or columns,
the same `ref` is reused throughout - this entails that the loop is
allocation-free, resulting in orders of magnitude faster loops. One should
pay attention not to hold to these references, because they will be mutated.

This means that - for instance - the following is correct:

```nim
let m = randomMatrix(1000, 1000)
var columnSum = zeros(1000)
for c in m.columns =
  columnSum += c
```

but the following will give wrong results (all elements of `cols` will be
identical at the end):

```nim
let m = randomMatrix(1000, 1000)
var cols = newSeq[Vector[float64]]()
for c in m.columns =
  cols.add(c)
```

If one needs a fresh reference for each element of the iteration, the
`rowsSlow` and `columnSlow` iterators are available, hence the
following modification is ok:

```nim
let m = randomMatrix(1000, 1000)
var cols = newSeq[Vector[float64]]()
for c in m.columnsSlow =
  cols.add(c)
```

#### Equality

There are two kinds of equality. The usual `==` operator will compare the
contents of vector and matrices exactly

```nim
let
  u = vector(1.0, 2.0, 3.0, 4.0)
  v = vector(1.0, 2.0, 3.0, 4.0)
  w = vector(1.0, 3.0, 3.0, 4.0)
u == v # true
u == w # false
```

Usually, though, one wants to take into account the errors introduced by
floating point operations. To do this, use the `=~` operator, or its
negation `!=~`:

```nim
let
  u = vector(1.0, 2.0, 3.0, 4.0)
  v = vector(1.0, 2.000000001, 2.99999999, 4.0)
u == v # false
u =~ v # true
```

#### Pretty-print

Both vectors and matrix have a pretty-print operation, so one can do

```nim
let m = randomMatrix(3, 7)
echo m8
```

and get something like

    [ [ 0.5024584865674662  0.0798945419892334  0.7512423051567048  0.9119041361916302  0.5868388894943912  0.3600554448403415  0.4419034543022882 ]
      [ 0.8225964245706265  0.01608615513584155 0.1442007939324697  0.7623388321096165  0.8419745686508193  0.08792951865247645 0.2902529012579151 ]
      [ 0.8488187232786935  0.422866666087792 0.1057975175658363  0.07968277822379832 0.7526946339452074  0.7698915909784674  0.02831893268471575 ] ]

#### Reshape operations

The following operations do not change the underlying memory layout of matrices
and vectors. This means they run in very little time even on big matrices, but
you have to pay attention when mutating matrices and vectors produced in this
way, since the underlying data is shared.

```nim
let
  m1 = randomMatrix(6, 9)
  m2 = randomMatrix(9, 6)
  v1 = randomVector(9)
echo m1.t # transpose, done in constant time without copying
echo m1 + m2.t
let m3 = m1.reshape(9, 6)
let m4 = v1.asMatrix(3, 3)
let v2 = m2.asVector
```

In case you need to allocate a copy of the original data, say in order to
transpose a matrix and then mutate the transpose without altering the original
matrix, a `clone` operation is available:

```nim
let m5 = m1.clone
```

Notice that `clone()` will be called internally anyway when using one of the
reshape operations with a matrix that is not contiguous (that is, a matrix
obtained by slicing).

There is also a hard transpose operation which, unlike `t()` will not try
to share storage but will always create a new matrix instead and copy the
data to the new matrix (this way, it will also preserve  the row-major or
colum-major order). The hard transpose is denoted `T()`, so that

```nim
m.t == m.T
```

always holds, although the internal representations differ.

#### BLAS Operations

A few linear algebra operations are available, wrapping BLAS libraries:

```nim
var v1 = randomVector(7)
let
  v2 = randomVector(7)
  m1 = randomMatrix(6, 9)
  m2 = randomMatrix(9, 7)
echo 3.5 * v1
v1 *= 2.3
echo v1 + v2
echo v1 - v2
echo v1 * v2 # dot product
echo v1 |*| v2 # Hadamard (component-wise) product
echo l_1(v1) # l_1 norm
echo l_2(v1) # l_2 norm
echo m2 * v1 # matrix-vector product
echo m1 * m2 # matrix-matrix product
echo m1 |*| m2 # Hadamard (component-wise) product
echo max(m1)
echo min(v2)
```

#### Universal functions

Universal functions are real-valued functions that are extended to vectors
and matrices by working element-wise. There are many common functions that are
implemented as universal functions:

```nim
sqrt
cbrt
log10
log2
log
exp
arccos
arcsin
arctan
cos
cosh
sin
sinh
tan
tanh
erf
erfc
lgamma
tgamma
trunc
floor
ceil
degToRad
radToDeg
```

This means that, for instance, the following check passes:

```nim
  let
    v1 = vector(1.0, 2.3, 4.5, 3.2, 5.4)
    v2 = log(v1)
    v3 = v1.map(log)

  assert v2 == v3
```

Universal functions work both on 32 and 64 bit precision, on vectors and
matrices.

If you have a function `f` of type `proc(x: float64): float64` you can use

```nim
makeUniversal(f)
```

to turn `f` into a (public) universal function. If you do not want to export
`f`, there is the equivalent template `makeUniversalLocal`.

#### Rewrite rules

A few rewrite rules allow to optimize a chain of linear algebra operations
into a single BLAS call. For instance, if you try

```nim
echo v1 + 5.3 * v2
```

this is not implemented as a scalar multiplication followed by a sum, but it
is turned into a single function call.

#### Solving linear systems

Some linear algebraic functions are included, currently for solving systems of
linear equations of the form `Ax = b`, for square matrices `A`. Functions to invert
square invertible matrices are also provided. These throw floating-point errors
in the case of non-invertible matrices.

These functions require a LAPACK implementation.

```nim
let
  a = randomMatrix(5, 5)
  b = randomVector(5)

echo solve(a, b)
echo a \ b # equivalent
echo a.inv()
```

#### Computing eigenvalues and eigenvectors

These functions require a LAPACK implementation.

To be documented.

#### Linking BLAS and LAPACK implementations

The library requires to link some BLAS implementation to perform the actual
linear algebra operations. By default, it tries to link whatever is the default
system-wide BLAS implementation.

A few compile flags are available to link specific BLAS implementations

    -d:atlas
    -d:openblas
    -d:mkl
    -d:mkl -d:threaded

Packages for various BLAS implementations are available from the package
managers of many Linux distributions. On OSX one can add the brew formulas
from [Homebrew Science](https://github.com/Homebrew/homebrew-science), such
as `brew install homebrew/science/openblas`.

You may also need to add suitable paths for the includes and library dirs.
On OSX, this should do the trick

```nim
switch("clibdir", "/usr/local/opt/openblas/lib")
switch("cincludes", "/usr/local/opt/openblas/include")
```

If you have problems with MKL, you may want to link it statically. Just pass
the options

```nim
--dynlibOverride:mkl_intel_lp64
--passL:${PATH_TO_MKL}/libmkl_intel_lp64.a
```

to enable static linking.

### Sparse linear algebra

To be documented.

## Working on the GPU

### Linking CUDA

It is possible to delegate work to the GPU using CUDA. The library has been
tested to work with NVIDIA CUDA 8.0, but it is possible that earlier
versions will work as well. In order to compile and link against CUDA, you
should make the appropriate headers and libraries available. If they are not
globally set, you can pass suitable options to the Nim compiler, such as

```
--cincludes:"/usr/local/cuda/include"
--clibdir:"/usr/local/cuda/lib64"
```

Support for CUDA is under the package `neo/cuda`, that needs to be imported
explicitly.

### Dense linear algebra

If you have a matrix or vector, you can move it on the GPU, and back
like this:

```nim
import neo, neo/cuda
let
  v = randomVector(12, max=1'f32)
  vOnTheGpu = v.gpu()
  vBackOnTheCpu = vOnTheGpu.cpu()
```

Vectors and matrices on the GPU support linear-algebraic operations via cuBLAS,
exactly like their CPU counterparts. A few operation - such as reading a single
element - are not supported, as it does not make much sense to copy a single
value back and forth from the GPU. Usually it is advisable to move vectors
and matrices to the GPU, make as many computations as possible there, and
finally move the result back to the CPU.

The following are all valid operations, assuming `v` and `w` are vectors on the
GPU, `m` and `n` are matrices on the GPU and the dimensions are compatible:

```nim
v * 3'f32
v + w
v -= w
m * v
m - n
m * n
```

For more information, look at the tests in `tests/cudadense`.

### Sparse linear algebra

To be documented.


## TODO

* Add support for matrices and vector on the stack
* Use rewrite rules to optimize complex operations into a single BLAS call
* More specialized BLAS operations
* Try on more platforms/configurations
* Make a proper benchmark
* Improve documentation
* Better pretty-print

## Contributing

Every contribution is very much appreciated! This can range from:

* using the library and reporting any issues and any configuration on which
  it works fine
* building other parts of the scientific environment on top of it
* writing blog posts and tutorials
* contributing actual code (see the **TODO** section)
