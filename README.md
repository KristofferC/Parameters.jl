# Parameters

[![Build Status](https://travis-ci.org/mauro3/Parameters.jl.svg?branch=master)](https://travis-ci.org/mauro3/Parameters.jl)
[![Build Status](https://ci.appveyor.com/api/projects/status/github/mauro3/Parameters.jl?branch=master&svg=true)](https://ci.appveyor.com/project/mauro3/parameters-jl/branch/master)

[![Parameters](http://pkg.julialang.org/badges/Parameters_0.3.svg)](http://pkg.julialang.org/?pkg=Parameters&ver=0.3)
[![Parameters](http://pkg.julialang.org/badges/Parameters_0.4.svg)](http://pkg.julialang.org/?pkg=Parameters&ver=0.4)


This is a package I use to handle numerical-model parameters, thus the
name.  However, it should be useful otherwise too.  Its main feature
is the macro `@with_kw` which decorates a type definition and creates:

- a keyword constructor for the type
- allows setting default values for the fields inside the type definition
- a constructor which allows creating a type-instance taking its defaults from
  another type instance
- packing and unpacking macros for the type: `@unpack_*` where `*` is
  the type name.
- generic packing and unpacking macros `@pack`, `@unpack` (work with
  any types).

The keyword-constructor and default-values functionality will probably
make it into Julia
([# 10146](https://github.com/JuliaLang/julia/issues/10146),
[#533](https://github.com/JuliaLang/julia/issues/5333) and
[#6122](https://github.com/JuliaLang/julia/pull/6122)) although
probably not with all the features present in this package.  I suspect
that this package should stay usable & useful even after this change
lands in Julia.  Note that keyword functions are currently slow in
Julia, so these constructors should not be used in hot inner loops.

[NEWS.md](https://github.com/mauro3/Parameters.jl/blob/master/NEWS.md)
keeps tabs on updates.

Manual by example:
```julia
using Parameters

abstract Paras{R<:Real, I<:Integer}

@with_kw immutable PhysicalPara{R} <: Paras{R}
    rw::R = 1000.
    ri::R = 900.
    L::R = 3.34e5
    g::R = 9.81
    cw::R = 4220.
    day::R = 24*3600.
end

# create an instance with the defaults
pp = PhysicalPara{Float64}()
# make another one based on the previous one with some modifications
pp2 = PhysicalPara(pp; cw=.11e-7, rw=100.)
# make one afresh with some non-defaults
pp3 = PhysicalPara{Float32}(cw=77, day= 987)

# Custom inner constructors:
@with_kw immutable MyS{R}
    a::R = 5
    b = 4
    
    # Can define inner constructors as long as:
    #  - one defining all positional arguments is given
    #  - no zero-positional arguments constructor is defined (as that
    #    would clash with the keyword constructor)
    #
    # Note that the keyword constructor goes through the positional
    # constructor, thus invariants defined there will be honored.

    MyS(a,b) = (@assert a>b; new(a,b))  # The keyword constructor
                                        # calls this constructor, so
                                        # the invariant is satisfied.
    MyS(a) = MyS{R}(a, a-1) # For this provide your own outer constructor:
end
MyS{R}(a::R) = MyS{R}(a)

MyS{Int}() # MyS(5,4)
ms = MyS(3) # MyS(3,2)
MyS(ms, b=-1) # MyS(3,-1)
try
    MyS(ms, b=6) # this will fail the assertion
end

# parameter interdependence
@with_kw immutable Para{R<:Real}
    a::R = 5
    b::R
    c::R = a+b
end
pa = Para{Int}(b=7)

## (Un)pack macros
#
# When working with parameters it is often convenient to unpack (and
# pack) all of them:
function fn(var, pa::Para)
    @unpack_Para pa # the macro is constructed during the @with_kw
                    # and called @unpack_*
    out = var + a + b
    b = 77
    @pack_Para pa
    return out, pa
end

out, pa = fn(7, pa)

# If only a few parameters are needed, or possibly in general, it is
# more prudent to be explicit which variables are introduced into the
# local scope:

function fn2(var, pa::Para)
    @unpack pa: a, b
    out = var + a + b
    b = 77
    @pack pa: b
    return out, pa
end

out, pa = fn1(7, pa)
```

This is [examples/ex1.jl](examples/ex1.jl).

# Warning

Note that the (un-)packing macros which unpack all fields have a few
pitfalls, as changing the type definition will change what local
variables are available in a function using `@unpack_*`.  Examples:

- adding a field `pi` to a type might hijack `Base.pi` usage in a
  function
- the `@unpack_*` will shadow an input argument
  of the function.  Which I found perplexing at times.

Thus it is probably better, in general, to use the `@(un)pack` macros.


# TODO

- do copy of fields on (re-)construct?
- think about mutables
