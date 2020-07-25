# [Frequently Asked Questions](@id faq)

This page is a compilation of frequently asked questions and answers.

## [Performance](@id faq_performance)

#### GPUs, multithreading and distributed computation support

Yes. The `*`DiffEq.jl libraries (OrdinaryDiffEq.jl, StochasticDiffEq.jl, and
DelayDiffEq.jl) are all written to be generic to the array and number types.
This means they will adopt the implementation that is given by the array type.
The in-place algorithms internally utilize Julia's broadcast (with some exceptions
due to a Julia bug for now, see [this issue](https://github.com/JuliaDiffEq/OrdinaryDiffEq.jl/issues/106))
and Julia's `mul!` in-place matrix multiplication function. The out-of-place
algorithms utilize standard arithmetical functions. Both additionally utilize
the user's norm specified via the common interface options and, if a stiff
solver, ForwardDiff/DiffEqDiffTools for the Jacobian calculation, and Base linear
factorizations for the linear solve. For your type, you may likely need to give
a [better form of the norm](@ref advanced_adaptive_stepsize_control),
[Jacobian](@ref performance_overloads),
or [linear solve calculations](@ref linear_nonlinear)
to fully utilize parallelism.

GPUArrays.jl (CuArrays.jl), ArrayFire.jl, DistributedArrays.jl have been tested and work in
various forms, where the last one is still not recommended for common use
yet.

The next question is whether it matters. Generally, your system has to be large
for parallelism to matter. Using a multithreaded array for broadcast we find
helpful around `N>1000`, though the Sundials manual says `N>100,000`. For high
order Runge-Kutta methods it's likely lower than the Sundials estimate because
of more operations packed into each internal step, but as always that will need
more benchmarks to be precise and will depend on the problem being solved. GPUs
generally require some intensive parallel operation in the user's `f` function
to be viable, for example a matrix multiplication for a stencil computation
in a PDE. If you're simply solving some ODE element-wise on a big array it likely
won't do much or it will slow things down just due to how GPUs work.
DistributedArrays require parallel linear solves to really matter, and thus are
only recommended when you have a problem that cannot fit into memory or are using
a stiff solver with a Krylov method for the linear solves.

#### My ODE is solving really slow

First, check for bugs. These solvers go through a ton of convergence tests and
so if there's a solver issue, it's either just something to do with how numerical
methods work or it's a user-error (generally the latter, though check the later
part of the FAQ on normal numerical errors). User-errors in the `f` function
causing a divergence of the solution is the most common reason for reported
slow codes.

If you have no bugs, great! The standard tricks for optimizing Julia code then
apply. What you want to do first is make sure your function does not allocate.
If your system is small (`<=100` ODEs/SDEs/DDEs/DAEs?), then you should set your
system up to use [StaticArrays.jl](https://github.com/JuliaArrays/StaticArrays.jl).
This is demonstrated
[in the ODE tutorial](@ref ode_other_types)
with static matrices. Static vectors/arrays are stack-allocated, and thus creating
new arrays is free and the compiler doesn't have to heap-allocate any of the
temporaries (that's the expensive part!). These have specialized super fast
dispatches for arithmetic operations and extra things like LU-factorizations,
and thus they are preferred when possible. However, they lose efficiency if they
grow too large.

For anything larger, you should use the `in-place` syntax `f(du,u,p,t)` and make
sure that your function doesn't allocate. Assuming you know of a `u0`, you
should be able to do:

```julia
du = similar(u0)
@time f(du,u0,p,t)
```

and see close to zero allocations and close to zero memory allocated. If you see
more, then you might have a type-instability or have temporary arrays. To find
type-instabilities, you should do:

```julia
@code_warntype f(du,u,p,t)
```

and read the printout to see if there's any types that aren't inferred by the
compiler, and fix them. If you have any global variables, you should make them
`const`. As for allocations, some common things that allocate
are:

- Array slicing, like `u[1:5]`. Instead, use `@view u[1:5]`
- Matrix multiplication with `*`. Instead of `A*b`, use `A_mul_B!(c,A,b)` for some
  pre-allocated cache vector `c`.
- Non-broadcasted expressions. Every expression on arrays should `.=` into another
  array, or it should be re-written to loop and do computations with scalar (or
  static array) values.

For an example of optimizing a function resulting from a PDE discretization, see
[this blog post](http://www.stochasticlifestyle.com/solving-systems-stochastic-pdes-using-gpus-julia/).

#### The stiff solver takes forever to take steps for my PDE discretization

The solvers for stiff solvers require solving a nonlinear equation each step.
In order to do so, they have to do a few Newton steps. By default, these methods
assume that the Jacobian is dense, automatically calculate the Jacobian for you,
and do a dense factorization. However, in many cases you may want to use alternatives
that are more tuned for your problem.

First of all, when available, it's recommended that you pass a function for computing
your Jacobian. This is discussed in the [performance overloads](@ref ode_explicit_jac)
section. Jacobians are especially helpful for Rosenbrock methods.

Secondly, if your Jacobian isn't dense, you shouldn't use a dense Jacobian! In
the Sundials algorithm you can set `linear_solver=:Band` for banded Jacobians
for example. More support is coming for this soon.

But lastly, you shouldn't use a dense factorization for large sparse matrices.
Instead, if you're using  a `*DiffEq` library you should
[specify a linear solver](@ref linear_nonlinear).
For Sundials.jl, you should change the `linear_solver` option. See
[the ODE solve Sundials portion](@ref ode_solve_sundials)
for details on that. Right now, Sundials.jl is the recommended method for stiff
problems with large sparse Jacobians. `linear_solver=:Band` should be used
if your Jacobian is banded and you can specify the band sizes. If you only
know the Jacobian is sparse, `linear_solver=:GMRES` is a good option. Once
again, a good reference for how to handle PDE discretizations can be found
[at this blog post](http://www.stochasticlifestyle.com/solving-systems-stochastic-pdes-using-gpus-julia/).

#### My Problem Has Discontinuities and is Unstable / Slow

[This Discourse post](https://discourse.julialang.org/t/handling-instability-when-solving-ode-problems/9019/5)
goes into detail for how to handle discontinuities in your ODE function and how
to use that extra information to speed up the solver.

## Complicated Models

#### Switching ODE functions in the middle of integration

There are a few ways to do this. The simplest way is to just have a parameter to
switch between the two. For example:

```julia
function f(du,u,p,t)
  if p == 0
    du[1] = 2u[1]
  else
    du[1] = -2u[1]
  end
  du[2] = -u[2]
end
```

Then in a callback you can make the `affect!` function modify `integrator.prob.p`.
For example, we can make it change when `u[2]<0.5` via:

```julia
condition(t,u,integrator) = u[2] - 0.5
affect!(integrator) = integrator.prob.p = 1
```

Then it will change betweeen the two ODE choices for `du1` at that moment.
Another way to do this is to make the ODE functions all be the same type
via FunctionWrappers.jl, but that is unnecessary. With the way that modern
processors work, there exists branch prediction and thus execution of a conditional
is free if it's predictable which branch will be taken. In this case, almost every
call to `f` takes the `p==0` route until the callback, at which point it is
almost always the `else` route. Therefore the processor will effectively get
rid of the computational cost associated with this, so you're likely over-optimizing
if you're going further (unless this change happens every step, but even then
this is probably the cheapest part of the computation...).

## Numerical Error

#### What does tolerance mean and how much error should I expect

The most useful options are the tolerances `abstol` and `reltol`. These tell the 
internal adaptive time stepping engine how precise of a solution you want. 
Generally, `reltol` is the relative accuracy while `abstol` is the accuracy when 
`u` is near zero. *These tolerances are local tolerances and thus are not global 
guarantees*. However, a good rule of thumb is that the total solution accuracy 
is 1-2 digits less than the relative tolerances. Thus for the defaults 
`abstol=1e-6` and `reltol=1e-3`, you can expect a global accuracy of about 1-2 
digits. This is standard across the board and applies to the native Julia methods, 
the wrapped Fortran and C++ methods, the calls to MATLAB/Python/R, etc.

#### The solver doesn't obey physical law X (e.g. conservation of energy)

Yes, this is because the numerical solution of the ODE is not the exact solution.
There are a few ways that you can handle this problem. One way is to get a more
exact solution. Thus instead of

```julia
sol = solve(prob,alg)
```

use

```julia
sol = solve(prob,alg,abstol=1e-10,reltol=1e-10)
```

Of course, there's always a tradeoff between accuracy and efficiency, so play
around to find out what's right for your problem.

Another thing you can do is use a callback. There are some
[premade callbacks in the callback library](@ref callback_library) which
handle these sorts of things like projecting to manifolds and preserving positivity.

##### Symplectic integrators don't conserve energy

Yes, symplectic integrators do not exactly conserve energy. It is a common
misconception that they do. What symplectic integrators actually do is solve
for a trajectory which rests on a symplectic manifold that is perturbed from
the true solution's manifold by the truncation error. This means that symplectic
integrators do not experience (very much) long time drift, but their orbit is
not exactly the same as the true solution in phase space and thus you will
see differences in energy that tend to look periodic. There is a small drift
which grows linearly and is related to floating point error, but this drift
is much less than standard methods. This is why symplectic methods are recommended
for long time integration.

For conserving energy, there are a few things you can do. First of all, the energy
error is related to the integration error, so simply solving with higher accuracy
will reduce the error. The results in the
[DiffEqBenchmarks](https://github.com/JuliaDiffEq/DiffEqBenchmarks.jl) show
that using a `DPRKN` method with low tolerance can be a great choice. Another
thing you can do is use
[the ManifoldProjection callback from the callback library](@ref callback_library).

#### How to get to zero error

You can't. For floating point numbers, you shouldn't use below `abstol=1e-14`
and `reltol=1e-14`. If you need lower than that, use arbitrary precision numbers
like BigFloats or [ArbFloats.jl](https://github.com/JuliaArbTypes/ArbFloats.jl).

## Autodifferentiation and Dual Numbers

#### Native Julia solvers compatibility with autodifferentiation


Yes, they are compatible with automatic differentiation! Take a look at the
[sensitivity analysis](@ref sensitivity)
page for more details.

If the algorithm does not have differentiation of parameter-depedendent events,
then you simply need to make the initial condition have elements of Dual numbers.
If the algorithm uses Dual numbers, you need to make sure that time is also
given by Dual numbers.

To show this in action, let's say we want to find the Jacobian of solution
of the Lotka-Volterra equation at `t=10` with respect to the parameters.

```julia
function func(du,u,p,t)
  du[1] = p[1] * u[1] - p[2] * u[1]*u[2]
  du[2] = -3 * u[2] + u[1]*u[2]
end
function f(p)
  prob = ODEProblem(func,eltype(p).([1.0,1.0]),(0.0,10.0),p)
  # Lower tolerances to show the methods converge to the same value
  solve(prob,Tsit5(),save_everystep=false,abstol=1e-12,reltol=1e-12)[end]
end
```

This function takes in new parameters and spits out the solution at the end.
We make the inital condition `eltype(p).([1.0,1.0])` so that way it's typed to
be Dual numbers whenever `p` is an array of `Dual` numbers, and we do the same
for the timespan just to show what you'd do if there was parameters-dependent events.
Then we can take the Jacobian via ForwardDiff.jl:

```julia
using ForwardDiff
ForwardDiff.jacobian(f,[1.5,1.0])

2×2 Array{Float64,2}:
  2.16056   0.188569
 -6.25677  -0.697978
```

and compare it to Calculus.jl:

```julia
Calculus.jacobian(f,[1.5,1.0],:central)

2×2 Array{Float64,2}:
  2.16056   0.188569
 -6.25677  -0.697978
```

#### I get Dual number errors when I solve my ODE with Rosenbrock or SDIRK methods

This is because you're using a cache which is not compatible with autodifferentiaion
via ForwardDiff.jl. For example, if we use the ODE function:

```julia
using LinearAlgebra, OrdinaryDiffEq
function foo(du, u, (A, tmp), t)
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end
prob = ODEProblem(foo, ones(5, 5), (0., 1.0), (ones(5,5), zeros(5,5)))
solve(prob, Rosenbrock23())
```

Here we use a cached temporary array in order to avoid the allocations of matrix
multiplication. When autodifferentiation occurs, the element type of `u` is
`Dual` numbers, so `A*u` produces `Dual` numbers, so the error arises when it
tries to write into `tmp`. There are two ways to avoid this. The first way,
the easy way, is to just turn off autodifferentiation with the `autodiff=false`
option in the solver. Every solver which uses autodifferentiation has this option.
Thus we'd solve this with:

```julia
prob = ODEProblem(f,rand(4),(0.0,1.0))
sol = solve(prob,Rosenbrock23(autodiff=false))
```

and it will use a numerical differentiation fallback (DiffEqDiffTools.jl) to
calculate Jacobians.

We could use `get_tmp` and `dualcache` functions from `DiffEqBase` to solve this issue, e.g.,

```julia
using LinearAlgebra, OrdinaryDiffEq
using DiffEqBase: get_tmp, dualcache
function foo(du, u, (A, tmp), t)
    tmp = DiffEqBase.get_tmp(tmp, u)
    mul!(tmp, A, u)
    @. du = u + tmp
    nothing
end
chunk_size = 5
prob = ODEProblem(foo, ones(5, 5), (0., 1.0), (ones(5,5), DiffEqBase.dualcache(zeros(5,5), Val{chunk_size})))
solve(prob, TRBDF2(chunk_size=chunk_size))
```

Note that one can adjust the chunk size by `DiffEqBase.dualcache(du, Val{N})`, where `N` is the chunk size.
