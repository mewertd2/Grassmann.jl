<p align="center">
  <img src="./docs/src/assets/logo.png" alt="Grassmann.jl"/>
</p>

# Grassmann.jl

*Static dual multivector types with graded-blade indexing, product algebra, and optional origin*

[![Build Status](https://travis-ci.org/chakravala/Grassmann.jl.svg?branch=master)](https://travis-ci.org/chakravala/Grassmann.jl)
[![Build status](https://ci.appveyor.com/api/projects/status/c36u0rgtm2rjcquk?svg=true)](https://ci.appveyor.com/project/chakravala/grassmann-jl)
[![Coverage Status](https://coveralls.io/repos/chakravala/Grassmann.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/chakravala/Grassmann.jl?branch=master)
[![codecov.io](http://codecov.io/github/chakravala/Grassmann.jl/coverage.svg?branch=master)](http://codecov.io/github/chakravala/Grassmann.jl?branch=master)

This package is a work in progress providing the necessary tools to work with arbitrary dual `MultiVector` elements with optional origin. Due to the parametric type system for the generating `VectorSpace`, the Julia compiler can fully preallocate and often cache values efficiently. Both static and mutable vector types are supported.

It is currently possible to do both high-performance numerical computations with `Grassmann` and it is also currently possible to use symbolic scalar coefficients when the `Reduce` package is also loaded (compatibility instructions at bottom).

### Requirements

This requires a merged version of `ComputedFieldTypes` at https://github.com/vtjnash/ComputedFieldTypes.jl

## Direct-sum yields `VectorSpace` parametric type polymorphism ⨁

Let `N` be the dimension of a `VectorSpace{N}`.
The metric signature of the `Basis{V,1}` elements of a vector space `V` can be specified with the `V"..."` constructor by using `+` and `-` to specify whether the `Basis{V,1}` element of the corresponding index squares to `+1` or `-1`.
For example, `V"+++"` constructs a positive definite 3-dimensional `VectorSpace`.
```Julia
julia> ℝ^3 == V"+++" == VectorSpace(3)
true
```
The direct sum operator `⊕` can be used to join spaces (alternatively `+`), and `'` is an involution which toggles a dual vector space with inverted signature.
```Julia
julia> V = ℝ'⊕ℝ^3
-+++

julia> V'
+---'

julia> W = V⊕V'
-++++---*
```
The direct sum of a `VectorSpace` and its dual `V⊕V'` represents the full mother space `V*`.
```Julia
julia> collect(V) # all vector basis elements
Grassmann.Algebra{-+++,16}(v, v₁, v₂, v₃, v₄, v₁₂, v₁₃, v₁₄, v₂₃, v₂₄, v₃₄, v₁₂₃, v₁₂₄, v₁₃₄, ...)

julia> collect(V') # all covector basis elements
Grassmann.Algebra{+---',16}(w, w¹, w², w³, w⁴, w¹², w¹³, w¹⁴, w²³, w²⁴, w³⁴, w¹²³, w¹²⁴, w¹³⁴, ...)

julia> collect(W) # all mixed basis elements
Grassmann.Algebra{-++++---*,256}(v, v₁, v₂, v₃, v₄, w¹, w², w³, w⁴, v₁₂, v₁₃, v₁₄, v₁w¹, v₁w², ...
```
Not to be confused with the *dual basis*, to generate *dual numbers* it is possible to specify a degenerate `Basis` element with `ϵ` (having property `vϵ*vϵ=0`) at the first index, i.e. `V"ϵ+++"`
Optionally, an additional element can be specified as the *origin* by using `o` instead, i.e. `V"o+++"` or `V"ϵo+++"`.
These two basis elements will be interpreted in the type system such that they propagate under transformations when combining a mixed dimension `Algebra` (provided the signature is compatible).

The `Dimension{N}` of the `Algebra` corresponds to the total number of `Basis{V,1}` vectors, but even though `V"ϵo+++"` of type `VectorSpace{5,1,1}` has 5 `Basis{V,1}` elements, it can be internally recognized in the direct sum algebra as being generated by a `3`-dimensional `VectorSpace{3,0,0}` due to the encoding of the degenerate `Basis` element with `D` and origin with `O` in the `VectorSpace{N,D,O}` type.

## Generating elements and geometric algebra Λ(V)

By virtue of Julia's multiple dispatch on the field type `T`, methods can specialize on the `Dimension{N}` and `Grade{G}` and `VectorSpace{N,D,O}` via the algebra's types, as well as `Basis{V,G}`, `SValue{V,G,B,T}`, `MValue{V,G,B,T}`, `SBlade{T,V,G}`, `MBlade{T,V,G}`, `MultiVector{T,V}`, and `MultiGrade{V}` types.

The elements of the `Algebra` can be generated in many ways using the `Basis` elements created by the `@basis` macro,
```Julia
julia> using Grassmann; @basis ℝ'⊕ℝ^3 # equivalent to basis"-+++"
(-+++, v, v₁, v₂, v₃, v₄, v₁₂, v₁₃, v₁₄, v₂₃, v₂₄, v₃₄, v₁₂₃, v₁₂₄, v₁₃₄, v₂₃₄, v₁₂₃₄)
```
As a result of this macro, all of the `Basis{V,G}` elements generated by that `VectorSpace` become available in the local workspace with the specified naming.
The first argument provides signature specifications, the second argument is the variable name for the `VectorSpace`, and the third and fourth argument are the the prefixes of the `Basis` vector names (and covector basis names). By default, `V` is assigned the `VectorSpace` and `v` is the prefix for the `Basis` elements. Thus,
```Julia
julia> V # Minkowski spacetime
-+++

julia> typeof(V) # dispatch by vector space
VectorSpace{4,0,0,0x0001,0}

julia> typeof(v13) # extensive type info
Basis{-+++,2,0x0005}

julia> v13∧v2 # exterior tensor product
-1v₁₂₃

julia> ans^2 # applies geometric product
1v
```
It is entirely possible to assign multiple different bases with different signatures without any problems. In the following command, the `@basis` macro arguments are used to assign the vector space name to `S` instead of `V` and basis elements to `b` instead of `v`, so that their local names do not interfere:
```Julia
julia> @basis "++++" S b;

julia> let k = (b1+b2)-b3
           for j ∈ 1:9
               k = k*(b234+b134)
               println(k)
       end end
0 + 1v₁₄ + 1v₂₄ + 2v₃₄
0 - 2v₁ - 2v₂ + 2v₃
0 - 2v₁₄ - 2v₂₄ - 4v₃₄
0 + 4v₁ + 4v₂ - 4v₃
0 + 4v₁₄ + 4v₂₄ + 8v₃₄
0 - 8v₁ - 8v₂ + 8v₃
0 - 8v₁₄ - 8v₂₄ - 16v₃₄
0 + 16v₁ + 16v₂ - 16v₃
0 + 16v₁₄ + 16v₂₄ + 32v₃₄
```
Alternatively, if you do not wish to assign these variables to your local workspace, the versatile `Grassmann.Algebra{N}` constructors can be used to contain them, which is exported to the user as the method `Λ(V)`,
```Julia
julia> G3 = Λ(3) # equivalent to Λ(V"+++"), Λ(ℝ^3), Λ.V3
Grassmann.Algebra{+++,8}(v, v₁, v₂, v₃, v₁₂, v₁₃, v₂₃, v₁₂₃)

julia> G3.v13 * G3.v12
v₂₃
```
Another way is `Λ.V3`, then it is possible to assign the **quaternion** generators `i,j,k` with
```Julia
julia> i,j,k = Λ.V3.v32, Λ.V3.v13, Λ.V3.v21
(-1v₂₃, v₁₃, -1v₁₂)

julia> @btime i^2, j^2, k^2, i*j*k
  158.925 ns (5 allocations: 112 bytes)
(-1v, -1v, -1v, -1v)
```
The parametric type formalism in `Grassmann` is highly expressive to enable the pre-allocation of geometric algebra computations for specific sparse-subalgebras, including the representation of rotational groups, Lie bivector algebras, and affine projective geometry.

### Reaching ∞ dimensions with `SparseAlgebra` and `ExtendedAlgebra`

It is possible to reach `Basis` elements up to `N=62` indices with `TensorAlgebra` having higher maximum dimensions than supported by Julia natively.
```Julia
julia> Λ(62)
[ Info: Extending thread-safe 4611686018427387904×Basis{VectorSpace{62,0,0,0},...}
Grassmann.ExtendedAlgebra{++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++,4611686018427387904}(v, ..., v₁₂₃₄₅₆₇₈₉₀abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ)

julia> Λ(62).v32a87Ng
-1v₂₃₇₈agN
```
The 62 indices require full alpha-numeric labeling with lower-case and capital letters. This now allows you to reach up to `4,611,686,018,427,387,904` dimensions with Julia `using Grassmann`. Then the volume element is
```Julia
v₁₂₃₄₅₆₇₈₉₀abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
```
However, Julia is only able to allocate full `MultiVector` for `N≤22`, with sparse operations only available at higher dimension.

While `Grassmann.Algebra{V}` is a container for the `TensorAlgebra` generators of `V`, the `Grassmann.Algebra` is only cached for `N≤8`.
For a `VectorSpace{N}` of dimension `8<N≤22`, the `Grassmann.SparseAlgebra` type is used.

```Julia
julia> Λ(22)
[ Info: Declaring thread-safe 4194304×Basis{VectorSpace{22,0,0,0},...}
Grassmann.SparseAlgebra{++++++++++++++++++++++,4194304}(v, ..., v₁₂₃₄₅₆₇₈₉₀abcdefghijkl)
```
This is the largest `SparseAlgebra` that can be generated with Julia, due to array size limitations.

To reach higher dimensions, for `N>22` the `Grassmann.ExtendedAlgebra` type is used.
It is suficient to work with a 64-bit representation (which is the default). And it turns out that with 61 standard keyboard characters, this fits nicely. Since 22 is the limit for the largest fully representable `MultiVector` with Julia, having a 64-bit representation still lets you extend to 44 generating `Basis` elements if you suddenly want to decide to have a dual vector space also.
```Julia
julia> V = ℝ^22
++++++++++++++++++++++

julia> Λ(V+V')
[ Info: Extending thread-safe 17592186044416×Basis{VectorSpace{44,0,0,17592181850112}*,...}
Grassmann.ExtendedAlgebra{++++++++++++++++++++++----------------------*,17592186044416}(v, ..., v₁₂₃₄₅₆₇₈₉₀abcdefghijklw¹²³⁴⁵⁶⁷⁸⁹⁰ABCDEFGHIJKL)
```
Currently, up to `N=62` is supported with alpha-numeric indexing. This is due to the defaults of the bit depth from the computer, so if you are 32-bit it is more limited.

At 22 dimensions and lower, you have better caching, and 8 dimensions or less have extra caching.
Thus, the largest Hilbert space that is fully reachable has 4,194,304 dimensions, but we can still reach out to 2,305,843,009,213,693,952 dimensions with the `ExtendedAlgebra` built in.
This is approximately `2^117` times smaller than the order of the Monster group. It is still feasible to extend to a further super-extended 128-bit representation using the `UInt128` type (but this will require further modifications of internals and helper functions.
To reach into infinity even further, it is theoretically possible to construct ultra-extensions also using dictionaries.
Full `MultiVector` elements are not representable when `ExtendedAlgebra` is used, but the performance of the `Basis` and sparse elements should be just as fast as for lower dimensions for the current `SubAlgebra` and `TensorAlgebra` types.
The sparse representations are a work in progress to be improved with time.

In order to work with multithreading using a `TensorAlgebra{V}`, it is necessary for it to be declared as thread-safe. This is usually done automatically by accessing it.
```Julia
julia> Λ(7) + Λ(7)'
[ Info: Allocating thread-safe 128×Basis{VectorSpace{7,0,0,0},...}
[ Info: Allocating thread-safe 128×Basis{VectorSpace{7,0,0,127}',...}
[ Info: Declaring thread-safe 16384×Basis{VectorSpace{14,0,0,16256}*,...}
Grassmann.SparseAlgebra{+++++++-------*,16384}(v, ..., v₁₂₃₄₅₆₇w¹²³⁴⁵⁶⁷)
```
One way of declaring all 3 combinations of a `VectorSpace{N}` and its dual is to ask for the sum `Λ(N) + Λ(N)'` or `Λ(V) + Λ(V)'`, which is equivalent to `Λ(V⊕V')`.

The staging of the precompilation and caching is designed so that a user can smoothly transition between very high dimensional and low dimensional algebras in a single session, with varying levels of extra caching and optimizations.

## Constructing linear transformations from mixed tensor product ⊗

Groups such as SU(n) can be represented with the dual Grassmann’s exterior product algebra, generating a `2^(2n)`-dimensional mother algebra with geometric product from the `n`-dimensional vector space and its dual vector space. The product of the vector basis and covector basis elements form the `n^2`-dimensional bivector subspace of the full `(2n)!/(2(2n−2)!)`-dimensional bivector sub-algebra.
The package `Grassmann` is working towards making the full extent of this number system available in Julia by using static compiled parametric type information to handle sparse sub-algebras, such as the (1,1)-tensor bivector algebra.

Note that `Λ.V3` gives the vector basis, and `Λ.C3` gives the covector basis:
```Julia
julia> Λ.V3
[ Info: Precomputing 8×Basis{VectorSpace{3,0,0,0},...}
Grassmann.Algebra{+++,8}(v, v₁, v₂, v₃, v₁₂, v₁₃, v₂₃, v₁₂₃)

julia> Λ.C3
[ Info: Precomputing 8×Basis{VectorSpace{3,0,0,7}',...}
Grassmann.Algebra{---',8}(w, w¹, w², w³, w¹², w¹³, w²³, w¹²³)
```
The following command yields a local 2D vector and covector basis,
```Julia
julia> mixedbasis"2"
(++--*, v, v₁, v₂, w¹, w², v₁₂, v₁w¹, v₁w², v₂w¹, v₂w², w¹², v₁₂w¹, v₁₂w², v₁w¹², v₂w¹², v₁₂w¹²)

julia> w1+2w2
0v₁ + 0v₂ + 1w¹ + 2w²

julia> ans(v1+v2)
3v
```
The sum `w1+2w2` is interpreted as a covector element of the dual vector space, which can be evaluated as a linear functional when a vector argument is input.
Using these in the workspace, it is possible to use the Grassmann exterior `∧`-tensor product operation to construct elements `ℒ` of the (1,1)-bivector subspace of linear transformations from the `Grade{2}` algebra.
```Julia
julia> ℒ = ((v1+2v2)∧(3w1+4w2))(2)
0v₁₂ + 3v₁w¹ + 4v₁w² + 6v₂w¹ + 8v₂w² + 0w¹²
```
The element `ℒ` is a linear form which can take `Grade{1}` vectors as input,
```Julia
julia> ℒ(v1+v2)
7v₁ + 14v₂ + 0w¹ + 0w²

julia> L = [1,2] * [3,4]'; L * [1,1]
2-element Array{Int64,1}:
  7
 14
```
which is a computation equivalent to a matrix computation.

This package is still a work in progress, and the API and implementation may change as more features and algebraic operations and product structure are added.

## Symbolic coefficients by declaring an alternative scalar algebra

```Julia
julia> using GaloisFields,Grassmann

julia> const F = GaloisField(7)
𝔽₇

julia> basis"2"
(++, v, v₁, v₂, v₁₂)

julia> @btime F(3)*v1
  21.076 ns (2 allocations: 32 bytes)
3v₁

julia> @btime inv($ans)
  26.636 ns (0 allocations: 0 bytes)
5v₁
```

Due to the abstract generality of the code generation of the `Grassmann` product algebra, it is easily possible to extend the entire set of operations to other kinds of scalar coefficient types.
By default, the coefficients are required to be `<:Number`. However, if this does not suit your needs, alternative scalar product algebras can be specified with
```Julia
generate_product_algebra(SymField,:(Sym.:*),:(Sym.:+),:(Sym.:-),:svec)
```
where `SymField` is the desired scalar field and `Sym` is the scope which contains the scalar field algebra for `SymField`.

Currently, with the use of `Requires` it is feasible to automatically enable symbolic scalar computation with [Reduce.jl](https://github.com/chakravala/Reduce.jl), e.g.
```Julia
julia> using Reduce, Grassmann
Reduce (Free CSL version, revision 4590), 11-May-18 ...
```
It is **essential** that the `Reduce` package was precompiled without the extra precompilation enabled if using a Linux operating system (`ENV["REDPRE"]=0`), since the extra precompilation causes a segfault when used with `StaticArrays`.
```Julia
julia> basis"2"
(++, v, v₁, v₂, v₁₂)

julia> (:a*v1 + :b*v2) ∧ (:c*v1 + :d*v2)
0.0 + (a * d - b * c)v₁₂

julia> (:a*v1 + :b*v2) * (:c*v1 + :d*v2)
a * c + b * d + (a * d - b * c)v₁₂
```
If these compatiblity steps are followed, then `Grassmann` will automatically declare the product algebra to use the `Reduce.Algebra` symbolic field operation scope.

It should be straight-forward to easily substitute any other extended algebraic operations and for extended fields and pull-requests are welcome.

## References
* C. Doran, D. Hestenes, F. Sommen, and N. Van Acker, [Lie groups as spin groups](http://geocalc.clas.asu.edu/pdf/LGasSG.pdf), J. Math Phys. (1993)
* David Hestenes, [Universal Geometric Algebra](http://lomont.org/Math/GeometricAlgebra/Universal%20Geometric%20Algebra%20-%20Hestenes%20-%201988.pdf), Pure and Applied (1988)
* Peter Woit, [Clifford algebras and spin groups](http://www.math.columbia.edu/~woit/LieGroups-2012/cliffalgsandspingroups.pdf), Lecture Notes (2012)
