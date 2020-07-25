# [ODE Solvers](@id ode_solve)

`solve(prob::ODEProblem,alg;kwargs)`

Solves the ODE defined by `prob` using the algorithm `alg`. If no algorithm is
given, a default algorithm will be chosen.

## Recommended Methods

It is suggested that you try choosing an algorithm using the `alg_hints`
keyword argument. However, in some cases you may want something specific,
or you may just be curious. This guide is to help you choose the right algorithm.

### Unknown Stiffness Problems

When the stiffness of the problem is unknown, it is recommended you use a
stiffness detection and auto-switching algorithm. These methods are multi-paradigm
and allow for efficient solution of both stiff and non-stiff problems. The cost
for auto-switching is very minimal but the choices are restrained and so they
are a good go-to method when applicable.

For default tolerances, `AutoTsit5(Rosenbrock23())` is a good choice. For lower
tolerances, using `AutoVern7` or `AutoVern9` with `Rodas4`, `KenCarp4`, or
`Rodas5` can all be good choices depending on the problem. For very large
systems (`>1000 ODEs?`), consider using `lsoda`.

### Non-Stiff Problems

For non-stiff problems, the native OrdinaryDiffEq.jl algorithms are vastly
more efficient than the other choices. For most non-stiff
problems, we recommend `Tsit5`. When more robust error control is required,
`BS5` is a good choice. If at moderate tolerances an the interpolation error
is very important, consider the `OwrenZen5` method. For fast solving at higher
tolerances, we recommend `BS3` (or `OwrenZen3` if the interpolation error is
important). For high accuracy but with the range of `Float64` (`~1e-8-1e-12`),
we recommend `Vern6`, `Vern7`, or `Vern8` as efficient choices.

For high accuracy non-stiff solving (`BigFloat` and tolerances like `<1e-12`),
we recommend the `Vern9` method. If a high-order method is needed with a high
order interpolant, then you should choose `Vern9` which is Order 9 with an
Order 9 interpolant. If you need extremely high accuracy (`<1e-30`?) and do
not need an interpolant, try the `Feagin12` or `Feagin14` methods. Note that the
Feagin methods are the only high-order optimized methods which do not include a
high-order interpolant (they do include a 3rd order Hermite interpolation if
needed). Note that these high order RK methods are more robust than the high order
Adams-Bashforth methods to discontinuities and achieve very high precision, and
are much more efficient than the extrapolation methods. However, the `VCABM`
method can be a good choice for high accuracy when the system of equations is
very large (`>1,000` ODEs?), the function calculation is very expensive,
or the solution is very smooth.

If strict error bounds are needed, then adaptive methods with defect controls
are required. Defect controls use an error measurement on the interpolating
polynomial to make the error estimate better capture the error over the full
interval. For medium accuracy calculations, `RK4` is a good choice.

### Stiff Problems

For stiff problems at high tolerances (`>1e-2`?) it is recommended that you use
`Rosenbrock23` or `TRBDF2`. These are robust to oscillations and massive
stiffness is needed, though are only efficient when low accuracy is needed.
`Rosenbrock23` is more efficient for small systems where re-evaluating and
re-factorizing the Jacobian is not too costly, and for sufficiently large
systems `TRBDF2` will be more efficient. `ABDF2` can be the most efficient
the largest systems or most expensive `f`.

At medium tolerances (`>1e-8`?) it is recommended you use `Rodas5`,
`Rodas4P` (the former is more efficient but the later is more reliable),
`Kvaerno5`, or `KenCarp4`. As native DifferentialEquations.jl solvers,
many Julia numeric types (such as BigFloats,
[ArbFloats](https://github.com/JuliaArbTypes/ArbFloats.jl), or
[DecFP](https://github.com/stevengj/DecFP.jl)) will work. When the equation is
defined via the `@ode_def` macro, these will be the most efficient.

For faster solving at low tolerances (`<1e-9`) but when `Vector{Float64}` is used,
use `radau`.

For asymptotically large systems of ODEs (`N>1000`?)
where `f` is very costly and the complex eigenvalues are minimal (low oscillations),
in that case `CVODE_BDF` will be the most efficient but requires `Vector{Float64}`.
`CVODE_BDF` will also do surprisingly well if the solution is smooth. However,
this method can handle less stiffness than other methods and its Newton iterations
may fail at low accuracy situations. Another good choice for this regime is `lsoda`.

#### Special Properties of Stiff Integrators

`ImplicitMidpoint` is a symmetric and symplectic integrator.
`Trapezoid` is a symmetric (almost symplectic) integrator with adaptive
timestepping. `ImplicitEuler` is an extension to the common algorithm with
adaptive timestepping and efficient quasi-Newton Jacobian re-usage which is fully
strong-stability presurving (SSP) for hyperbolic PDEs.

Notice that `Rodas4` loses accuracy on discretizations of nonlinear
parabolic PDEs, and thus it's suggested you replace it with `Rodas4P` in those
situations which is 3rd order. `ROS3P` is only third order and achieves 3rd order
on such problems and can thus be more efficient in this case.

## Translations from MATLAB/Python/R

For users familiar with MATLAB/Python/R, good translations of the standard
library methods are as follows:

- `ode23` --> `BS3()`
- `ode45`/`dopri5` --> `DP5()`, though in most cases `Tsit5()` is more efficient
- `ode23s` --> `Rosenbrock23()`, though in most cases `Rodas4()` is more efficient
- `ode113` --> `VCABM()`, though in many cases `Vern7()` is more efficient
- `dop853` --> `DP8()`, though in most cases `Vern7()` is more efficient
- `ode15s`/`vode` --> `QNDF()`, though in many cases `CVODE_BDF()`, `Rodas4()`,
  `KenCarp4()`, `TRBDF2()`, or `RadauIIA()` are more efficient
- `ode23t` --> `Trapezoid()`
- `ode23tb` --> `TRBDF2()`
- `lsoda` --> `lsoda()`, though `AutoTsit5(Rosenbrock23())` or `AutoVern7(Rodas5())`
  may be more efficient. Note that `lsoda()` requires the LSODA.jl extension, which
  can be added via `]add LSODA; using LSODA`.
- `ode15i` --> `IDA()`, though in many cases `Rodas4()` can handle the DAE and is
  significantly more efficient

## Full List of Methods

### OrdinaryDiffEq.jl for Non-Stiff Equations

Unless otherwise specified, the OrdinaryDiffEq algorithms all come with a
3rd order Hermite polynomial interpolation. The algorithms denoted as having a
"free" interpolation means that no extra steps are required for the
interpolation. For the non-free higher order interpolating functions, the extra
steps are computed lazily (i.e. not during the solve).

The OrdinaryDiffEq.jl algorithms achieve the highest performance for non-stiff
equations while being the most generic: accepting the most Julia-based types,
allow for sophisticated event handling, etc. On stiff ODEs these algorithms
again consistently among the top. OrdinaryDiffEq.jl is recommended for most ODE
problems.

#### Linear Methods

##### u' = A(t)u solvers

- `MagnusMidpoint` - Second order Magnus Midpoint method.
- `MagnusLeapfrog`- Second order Magnus Leapfrog method.
- `MagnusGauss4` - Fourth order Magnus method approximated using a two stage Gauss quadrature.
- `MagnusGL4`- Fourth order Magnus method approximated using Gauss-Legendre quadrature.
- `MagnusNC6`- Sixth order Magnus method approximated using Newton-Cotes quadrature.
- `MagnusGL6`- Sixth order Magnus method approximated using Gauss-Legendre quadrature.
- `MagnusNC8`- Eighth order Magnus method approximated using Newton-Cotes quadrature.
- `MagnusGL8`- Eighth order Magnus method approximated using Gauss-Legendre quadrature.

Example:

```julia
function update_func(A,u,p,t)
    A[1,1] = cos(t)
    A[2,1] = sin(t)
    A[1,2] = -sin(t)
    A[2,2] = cos(t)
end
A = DiffEqArrayOperator(ones(2,2),update_func=update_func)
prob = ODEProblem(A, ones(2), (1.0, 6.0))
sol = solve(prob,MagnusGL6(),dt=1/10)
```

#### Explicit Runge-Kutta Methods

- `Euler`- The canonical forward Euler method. Fixed timestep only.
- `Midpoint` - The second order midpoint method. Uses embedded Euler method for
  adaptivity.
- `Heun` - The second order Heun's method. Uses embedded Euler method for
  adaptivity.
- `Ralston` - The optimized second order midpoint method. Uses embedded Euler.
  method for adaptivity.
- `RK4` - The canonical Runge-Kutta Order 4 method. Uses a defect control for
  adaptive stepping using maximum error over the whole interval.
- `BS3` - Bogacki-Shampine 3/2 method.
- `OwrenZen3` - Owren-Zennaro optimized interpolantion 3/2 method (free 3th
  order interpolant).
- `OwrenZen4` - Owren-Zennaro optimized interpolantion 4/3 method (free 4th
  order interpolant).
- `OwrenZen5` - Owren-Zennaro optimized interpolantion 5/4 method (free 5th
  order interpolant).
- `DP5` - Dormand-Prince's 5/4 Runge-Kutta method. (free 4th order interpolant).
- `Tsit5` - Tsitouras 5/4 Runge-Kutta method. (free 4th order interpolant).
- `Anas5(w)` - 4th order Runge-Kutta method designed for periodic problems.
  Requires a periodicity estimate `w` which when accurate the method becomes
  5th order (and is otherwise 4th order with less error for better estimates).
- `FRK65(w=0)` - Zero Dissipation Runge-Kutta of 6th order. Takes an optional
  argument `w` to for the periodicity phase, in which case this method results in
  zero numerical dissipation.
- `PFRK87(w=0)` - Phase-fitted Runge-Kutta Runge-Kutta of 8th order. Takes an optional
  argument `w` to for the periodicity phase, in which case this method results in
  zero numerical dissipation.
- `RKO65` - Olver's 6 stage 5th order method. This method is robust on problems
  which have a singularity at `t=0`.
- `TanYam7` - Tanaka-Yamashita 7 Runge-Kutta method.
- `DP8` - Hairer's 8/5/3 adaption of the Dormand-Prince Runge-Kutta method.
  (7th order interpolant).
- `TsitPap8` - Tsitouras-Papakostas 8/7 Runge-Kutta method.
- `Feagin10` - Feagin's 10th-order Runge-Kutta method.
- `Feagin12` - Feagin's 12th-order Runge-Kutta method.
- `Feagin14` - Feagin's 14th-order Runge-Kutta method.

Example usage:

```julia
alg = Tsit5()
solve(prob,alg)  
```

Additionally, the following algorithms have a lazy interpolant:

- `BS5` - Bogacki-Shampine 5/4 Runge-Kutta method. (lazy 5th order interpolant).
- `Vern6` - Verner's "Most Efficient" 6/5 Runge-Kutta method. (lazy 6th order
  interpolant).
- `Vern7` - Verner's "Most Efficient" 7/6 Runge-Kutta method. (lazy 7th order
  interpolant).
- `Vern8` - Verner's "Most Efficient" 8/7 Runge-Kutta method. (lazy 8th order
  interpolant)
- `Vern9` - Verner's "Most Efficient" 9/8 Runge-Kutta method. (lazy 9th order
  interpolant)

These methods require a few extra steps in order to compute the high order
interpolation, but these steps are only taken when the interpolation is used.
These methods when lazy assume that the parameter vector `p` will be unchanged
between the moment of the interval solving and the interpolation. If `p` is
changed in a ContinuousCallback, or in a DiscreteCallback and the continuous
solution is used after the full solution, then set `lazy=false`.

Example:

```julia
solve(prob,Vern7()) # lazy by default
solve(prob,Vern7(lazy=false))
```

#### Parallel Explicit Runge-Kutta Methods

- `KuttaPRK2p5` - A 5 parallel, 2 processor explicit Runge-Kutta method of 5th order.

These methods utilize multithreading on the `f` calls to parallelize the problem. This
requires that simultaneous calls to `f` are thread-safe.

#### Explicit Strong-Stability Preserving Runge-Kutta Methods for Hyperbolic PDEs (Conservation Laws)

- `SSPRK22` - The two-stage, second order strong stability preserving (SSP)
  method of Shu and Osher (SSP coefficient 1, free 2nd order SSP interpolant).
  Fixed timestep only.
- `SSPRK33` - The three-stage, third order strong stability preserving (SSP)
  method of Shu and Osher (SSP coefficient 1, free 2nd order SSP interpolant).
  Fixed timestep only.
- `SSPRK53` - The five-stage, third order strong stability preserving (SSP)
  method of Ruuth (SSP coefficient 2.65, free 3rd order Hermite interpolant).
  Fixed timestep only.
- `SSPRK63` - The six-stage, third order strong stability preserving (SSP)
  method of Ruuth (SSP coefficient 3.518, free 3rd order Hermite interpolant).
  Fixed timestep only.
- `SSPRK73` - The seven-stage, third order strong stability preserving (SSP)
  method of Ruuth (SSP coefficient 4.2879, free 3rd order Hermite interpolant). Fixed timestep only.
- `SSPRK83` - The eight-stage, third order strong stability preserving (SSP)
  method of Ruuth (SSP coefficient 5.107, free 3rd order Hermite interpolant).
  Fixed timestep only.
- `SSPRK432` - A  3/2 adaptive strong stability preserving (SSP) method with
  five stages (SSP coefficient 2, free 2nd order SSP interpolant).
- `SSPRK932` - A  3/2 adaptive strong stability preserving (SSP) method with
  nine stages (SSP coefficient 6, free 3rd order Hermite interpolant).
- `SSPRK54` - The five-stage, fourth order strong stability preserving (SSP)
  method of Spiteri and Ruuth (SSP coefficient 1.508, 3rd order Hermite
  interpolant). Fixed timestep only.
- `SSPRK104` - The ten-stage, fourth order strong stability preserving method
  of Ketcheson (SSP coefficient 6, free 3rd order Hermite interpolant).
  Fixed timestep only.
- `SSPRKMSVS32` - 3-stage, 2nd order SSP-optimal linear multistep method.
  (SSP coefficent 0.5, 3rd order Hermite interpolant). Fixed timestep only.
- `SSPRKMSVS43` - 4-stage, 3rd order SSP-optimal linear multistep method.
  (SSP coefficent 0.33, 3rd order Hermite interpolant). Fixed timestep only.

The SSP coefficients of the methods can be queried as `ssp_coefficient(alg)`.
All explicit SSP methods take two optional arguments
`SSPXY(stage_limiter!, step_limiter!)`, where `stage_limiter!` and `step_limiter`
are functions taking arguments of the form `limiter!(u, f, t)`. Here, `u` is the
new solution value (updated inplace) after an explicit Euler stage / the whole
time step , `f` the time derivative function (semidiscretisation for PDEs), and
`t` the current time. These limiters can be used to enforce physical constraints,
e.g. the positivity preserving limiters of Zhang and Shu (Zhang, Xiangxiong, and
Chi-Wang Shu. "Maximum-principle-satisfying and positivity-preserving high-order
schemes for conservation laws: survey and new developments." Proceedings of the
Royal Society of London A: Mathematical, Physical and Engineering Sciences. The
Royal Society, 2011.).

#### Low-Storage Methods

- `ORK256` - 5-stage, second order low-storage method for wave propogation
  equations. Fixed timestep only.
- `SSPRK53_2N1` and `SSPRK53_2N2` - 5-stage, third order low-storage methods
  with large SSP coefficients. (SSP coefficient 2.18 and 2.15, free 3rd order
  Hermite interpolant). Fixed timestep only.
- `CarpenterKennedy2N54` - The five-stage, fourth order low-storage method of Carpenter and Kennedy
  (free 3rd order Hermite interpolant). Fixed timestep only. Designed for hyperbolic PDEs (stability properties).
- `NDBLSRK124` - 12-stage, fourth order low-storage method with optimized
  stability regions for advection-dominated problems. Fixed timestep only.
- `NDBLSRK134` - 13-stage, fourth order low-storage method with optimized
  stability regions for advection-dominated problems. Fixed timestep only.
- `NDBLSRK144` - 14-stage, fourth order low-storage method with optimized
  stability regions for advection-dominated problems. Fixed timestep only.
- `CFRLDDRK64` - 6-stage, fourth order low-storage, low-dissipation,
  low-dispersion scheme. Fixed timestep only.
- `TSLDDRK74` - 7-stage, fourth order low-storage low-dissipation,
  low-dispersion scheme with maximal accuracy and stability limit
  along the imaginary axes. Fixed timestep only.
- `DGLDDRK73_C` - 7-stage, third order low-storage low-dissipation,
  low-dispersion scheme for discontinuous Galerkin space discretizations
  applied to wave propagation problems, optimized for PDE discretizations
  when maximum spatial step is small due to geometric features of computational
  domain. Fixed timestep only.
- `DGLDDRK84_C` - 8-stage, fourth order low-storage low-dissipation,
  low-dispersion scheme for discontinuous Galerkin space discretizations
  applied to wave propagation problems, optimized for PDE discretizations
  when maximum spatial step is small due to geometric features of computational
  domain. Fixed timestep only.
- `DGLDDRK84_F` - 8-stage, fourth order low-storage low-dissipation,
  low-dispersion scheme for discontinuous Galerkin space discretizations
  applied to wave propagation problems, optimized for PDE discretizations
  when the maximum spatial step size is not constrained. Fixed timestep only.
- `HSLDDRK64` - 6-stage, fourth order low-stage, low-dissipation, low-dispersion
  scheme. Fixed timestep only.
- `RK46NL` - 6-stage, fourth order low-stage, low-dissipation, low-dispersion
  scheme. Fixed timestep only.
- `ParsaniKetchesonDeconinck3S32` - 3-stage, second order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S82` - 8-stage, second order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S53` - 5-stage, third order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S173` - 17-stage, third order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S94` - 9-stage, fourth order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S184` - 18-stage, fourth order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S105` - 10-stage, fifth order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `ParsaniKetchesonDeconinck3S205` - 20-stage, fifth order (3S) low-storage scheme, optimised for for the
  spectral difference method applied to wave propagation problems.
- `CKLLSRK43_2` - 4-stage, third order low-storage scheme, optimised for compressible Navier–Stokes equations..
- `CKLLSRK54_3C` - 5-stage, fourth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK95_4S` - 9-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK95_4C` - 9-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK95_4M` - 9-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK54_3C_3R` - 5-stage, fourth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK54_3M_3R` - 5-stage, fourth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK54_3N_3R` - 5-stage, fourth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK85_4C_3R` - 8-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK85_4M_3R` - 8-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK85_4P_3R` - 8-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK54_3N_4R` - 5-stage, fourth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK54_3M_4R` - 5-stage, fourth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK65_4M_4R` - 6-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK85_4FM_4R` - 8-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.
- `CKLLSRK75_4M_5R` - 7-stage, fifth order low-storage scheme, optimised for compressible Navier–Stokes equations.

__NOTE__: All the 2N Methods (`ORK256`, `CarpenterKennedy2N54`, `NDBLSRK124`, `NDBLSRK134`, `NDBLSRK144`, `DGLDDRK73_C`, `DGLDDRK84_C`, `DGLDDRK84_F` and `HSLDDRK64`) work on the basic principle of being able to perform step `S1 = S1 + F(S2)` in just 2 registers. Certain optimizations have been done to achieve this theoritical limit (when `alias_u0` is set) but have a limitation that `du` should always be on the left hand side (assignments only) in the implementation.

Example - This is an invalid implementation for 2N methods:

```julia
function f(du,u,p,t)
  du[1] = u[1] * u[2]
  du[2] = du[1] * u[2] # du appears on the RHS
end
```

If you don't wish to have the optimization and have to use `du` on the RHS, please set the keyword argument `williamson_condition` to `false` in the algorithm (by default it is set to `true`). In this case 3 registers worth memory would be needed instead.

Example :

```julia
alg = CarpenterKennedy2N54(;williamson_condition=false)
```

So the above implementation of `f` becomes valid.

#### Parallelized Explicit Extrapolation Methods

The following are adaptive order, adaptive step size extrapolation methods:

- `AitkenNeville` - Euler extrapolation using Aitken-Neville with the Romberg Sequence.
- `ExtrapolationMidpointDeuflhard` - Midpoint extrapolation using Barycentric coordinates
- `ExtrapolationMidpointHairerWanner` - Midpoint extrapolation using Barycentric coordinates,
  following Hairer's `ODEX` in the adaptivity behavior.

These methods have arguments for `max_order`, `min_order`, and `init_order` on the adaptive order
algorithm. `threading` denotes whether to automatically multithread the `f` evaluations,
allowing for a high degree of within-method parallelism. The defaults are:

- `max_order=10`
- `min_order=1` except for `ExtrapolationMidpointHairerWanner` it's 2.
- `init_order=5`
- `threading=true`

Additionally, the `ExtrapolationMidpointDeuflhard` and `ExtrapolationMidpointHairerWanner`
methods have the additional argument:

* `sequence`: the step-number sequences, also called the subdividing
 sequence. Possible values are `:harmonic`, `:romberg` or `:bulirsch`. Default
 is `:harmonic`.

To override, utilize the keyword arguments. For example:

```julia
alg = ExtrapolationMidpointDeuflhard(max_order=7,min_order=4,init_order=4,sequence=:bulirsch,threading=false)
solve(prob,alg)
```

Note that the order that is referred to is the extrapolation order. For `AitkenNeville`
this is the order of the method, for the others an extrapolation order of `n`
gives an order `2(n+1)` method.

#### Explicit Multistep Methods

Methods using the approximation at more than one previous mesh point to determine
the approximation at the next point are called multistep methods. These methods
tend to be more efficient as the size of the system or the cost of `f` increases.

#### Adams-Bashforth Explicit Methods

These methods require a choice of `dt`.

- `AB3` - The 3-step third order multistep method. Ralston's Second Order Method
  is used to calculate starting values.
- `AB4` - The 4-step fourth order multistep method. Runge-Kutta method of order
  4 is used to calculate starting values.  
- `AB5` - The 5-step fifth order multistep method. Runge-Kutta method of order
  4 is used to calculate starting values.  
- `ABM32` - It is third order method. In `ABM32`, `AB3` works as predictor and
  Adams Moulton 2-steps method works as Corrector. Ralston's Second Order Method
  is used to calculate starting values.  
- `ABM43` - It is fourth order method. In `ABM43`, `AB4` works as predictor and
  Adams Moulton 3-steps method works as Corrector. Runge-Kutta method of order
  4 is used to calculate starting values.  
- `ABM54` - It is fifth order method. In `ABM54`, `AB5` works as predictor and
  Adams Moulton 4-steps method works as Corrector. Runge-Kutta method of order 4
  is used to calculate starting values.

#### Adaptive step size Adams explicit Methods

- `VCAB3` - The 3rd order Adams method. Bogacki-Shampine 3/2 method is used to
  calculate starting values.  
- `VCAB4` - The 4th order Adams method. Runge-Kutta 4 is used to calculate
  starting values.  
- `VCAB5` - The 5th order Adams method. Runge-Kutta 4 is used to calculate
  starting values.
- `VCABM3` - The 3rd order Adams-Moulton method. Bogacki-Shampine 3/2 method is used
  to calculate starting values.  
- `VCABM4` - The 4th order Adams-Moulton method. Runge-Kutta 4 is used to calculate
  starting values.  
- `VCABM5` - The 5th order Adams-Moulton method. Runge-Kutta 4 is used to calculate
  starting values.
- `VCABM` - An adaptive order adaptive time Adams Moulton method. It uses an
  order adaptivity algorithm is derived from Shampine's DDEABM.
- `AN5` - An adaptive 5th order fixed-leading coefficient Adams method in
  Nordsieck form.
- `JVODE_Adams` - An adaptive time adaptive order fixed-leading coefficient Adams
  method in Nordsieck form. The order adaptivity algorithm is derived from
  Sundials' `CVODE_Adams`. In development.

### OrdinaryDiffEq.jl for Stiff Equations

#### SDIRK Methods

- `ImplicitEuler` - A 1st order implicit solver. A-B-L-stable. Adaptive
  timestepping through a divided differences estimate via memory. Strong-stability
  preserving (SSP).
- `ImplicitMidpoint` - A second order A-stable symplectic and symmetric implicit
  solver. Good for highly stiff equations which need symplectic integration.
- `Trapezoid` - A second order A-stable symmetric ESDIRK method. "Almost
  symplectic" without numerical dampening. Also known as Crank-Nicolson when
  applied to PDEs. Adaptive timestepping via divided differences on the memory.
  Good for highly stiff equations which are non-oscillatory.
- `TRBDF2` - A second order A-B-L-S-stable one-step ESDIRK method. Includes
  stiffness-robust error estimates for accurate adaptive timestepping, smoothed
  derivatives for highly stiff and oscillatory problems.
- `SDIRK2` - An A-B-L stable 2nd order SDIRK method
- `Kvaerno3` - An A-L stable stiffly-accurate 3rd order ESDIRK method
- `KenCarp3` - An A-L stable stiffly-accurate 3rd order ESDIRK method with splitting
- `Cash4` - An A-L stable 4th order SDIRK method
- `Hairer4` - An A-L stable 4th order SDIRK method
- `Hairer42` - An A-L stable 4th order SDIRK method
- `Kvaerno4` - An A-L stable stiffly-accurate 4th order ESDIRK method
- `KenCarp4` - An A-L stable stiffly-accurate 4th order ESDIRK method with splitting
- `KenCarp47` - An A-L stable stiffly-accurate 4th order seven-stage ESDIRK method with splitting
- `Kvaerno5` - An A-L stable stiffly-accurate 5th order ESDIRK method
- `KenCarp5` - An A-L stable stiffly-accurate 5th order ESDIRK method with splitting
- `KenCarp58` - An A-L stable stiffly-accurate 5th order eight-stage ESDIRK method with splitting

#### Fully-Implicit Runge-Kutta Methods (FIRK)

- `RadauIIA3` - An A-B-L stable fully implicit Runge-Kutta method with internal
  tableau complex basis transform for efficiency.
- `RadauIIA5` - An A-B-L stable fully implicit Runge-Kutta method with internal
  tableau complex basis transform for efficiency.

#### Parallel Diagonally Implicit Runge-Kutta Methods

- `PDIRK44` - A 2 processor 4th order diagonally non-adaptive implicit method.

These methods also have option `nlsolve` same as SDIRK methods. These methods also need `f` to be thread safe. It parallelises the `nlsolve` calls inside the method.  

#### Rosenbrock Methods


- `ROS3P` - 3rd order A-stable and stiffly stable Rosenbrock method. Keeps high
  accuracy on discretizations of nonlinear parabolic PDEs.
- `Rodas3` - 3rd order A-stable and stiffly stable Rosenbrock method.
- `RosShamp4`- An A-stable 4th order Rosenbrock method.
- `Veldd4` - A 4th order D-stable Rosenbrock method.
- `Velds4` - A 4th order A-stable Rosenbrock method.
- `GRK4T` - An efficient 4th order Rosenbrock method.
- `GRK4A` - An A-stable 4th order Rosenbrock method. Essentially "anti-L-stable"
  but efficient.
- `Ros4LStab` - A 4th order L-stable Rosenbrock method.
- `Rodas4` - A 4th order A-stable stiffly stable Rosenbrock method with a
  stiff-aware 3rd order interpolant
- `Rodas42` - A 4th order A-stable stiffly stable Rosenbrock method with a
  stiff-aware 3rd order interpolant
- `Rodas4P` - A 4th order A-stable stiffly stable Rosenbrock method with a
  stiff-aware 3rd order interpolant. 4th order on linear parabolic problems and
  3rd order accurate on nonlinear parabolic problems (as opposed to lower if not
  corrected).
- `Rodas5` - A 5th order A-stable stiffly stable Rosenbrock method. Currently has
  a Hermite interpolant because its stiff-aware 3rd order interpolant is not
  yet implemented.

#### Rosenbrock-W Methods

- `Rosenbrock23` - An Order 2/3 L-Stable Rosenbrock-W method which is good for very
  stiff equations with oscillations at low tolerances. 2nd order stiff-aware
  interpolation.
- `Rosenbrock32` - An Order 3/2 A-Stable Rosenbrock-W method which is good for mildy
  stiff equations without oscillations at low tolerances. Note that this method
  is prone to instability in the presence of oscillations, so use with caution.
  2nd order stiff-aware interpolation.
- `RosenbrockW6S4OS` - A 4th order L-stable Rosenbrock-W method (fixed step only).
- `ROS34PW1a` - A 4th order L-stable Rosenbrock-W method.
- `ROS34PW1b` - A 4th order L-stable Rosenbrock-W method.
- `ROS34PW2` - A 4th order stiffy accurate Rosenbrock-W method for PDAEs.
- `ROS34PW3` - A 4th order strongly A-stable (Rinf~0.63) Rosenbrock-W method.

#### Stabilized Explicit Methods

- `ROCK2` - Second order stabilized Runge-Kutta method. Exhibits high stability
  for real eigenvalues and is smoothened to allow for moderate sized complex
  eigenvalues.
- `ROCK4` - Fourth order stabilized Runge-Kutta method. Exhibits high stability
  for real eigenvalues and is smoothened to allow for moderate sized complex
  eigenvalues.
- `RKC` - Second order stabilized Runge-Kutta method. Exhibits high stability
  for real eigenvalues and is smoothened to allow for moderate sized complex
  eigenvalues.
- `SERK2v2` - Second order stabilized extrapolated Runge-Kutta method. Exhibits
  high stability for real eigenvalues and is smoothened to allow for moderate
  sized complex eigenvalues.
- `ESERK5` - Fifth order stabilized extrapolated Runge-Kutta method. Exhibits
  high stability for real eigenvalues and is smoothened to allow for moderate
  sized complex eigenvalues.

ROCK methods offer a `min_stages` and `max_stages` functionality. SERK methods
derive higher orders by Aitken-Neville algorithm. SERK2v2 is defaulted to Predictive
control but has option of PI control.

#### Parallelized Implicit Extrapolation Methods

The following are adaptive order, adaptive step size extrapolation methods:

- `ImplicitEulerExtrapolation` - Extrapolation of implicit Euler method with Romberg sequence.
  Similar to Hairer's `SEULEX`.
- `ImplicitDeuflhardExtrapolation` - Midpoint extrapolation using Barycentric coordinates
- `ImplicitHairerWannerExtrapolation` - Midpoint extrapolation using Barycentric coordinates,
  following Hairer's `SODEX` in the adaptivity behavior.

These methods have arguments for `max_order`, `min_order`, and `init_order` on the adaptive order
algorithm. `threading` denotes whether to automatically multithread the `f` evaluations
and J/W instantiations+factorizations, allowing for a high degree of
within-method parallelism. The defaults are:

- `max_order=10`
- `min_order=1` except for `ImplicitHairerWannerExtrapolation` it's 2.
- `init_order=5`
- `threading=true`

Additionally, the `ImplicitDeuflhardExtrapolation` and `ImplicitHairerWannerExtrapolation`
methods have the additional argument:

* `sequence`: the step-number sequences, also called the subdividing
 sequence. Possible values are `:harmonic`, `:romberg` or `:bulirsch`. Default
 is `:harmonic`.

To override, utilize the keyword arguments. For example:

```julia
alg = ImplicitDeuflhardExtrapolation(max_order=7,min_order=4,init_order=4,sequence=:bulirsch)
solve(prob,alg)
```

Note that the order that is referred to is the extrapolation order. For `ImplicitEulerExtrapolation`
this is the order of the method, for the others an extrapolation order of `n`
gives an order `2(n+1)` method.

#### Parallelized DIRK Methods

These methods parallelize the J/W instantiation and factorization, making them
efficient on small highly stiff ODEs. Has an option `threading=true` to turn
on/off multithreading.

- `PDIRK44`: a 4th order 2-processor DIRK method.

#### Exponential Methods for Linear and Affine Problems

- `LinearExponential` - Exact solution formula for linear, time-independent problems.
  Expects the right hand side function to be a
  [`AbstractDiffEqOperator`](@ref).

Options:

- `krylov` - symbol. One of
  - :off (default) - cache the operator beforehand. Requires `Matrix(A)` method
    defined for the operator `A`.
  - :simple - uses simple Krylov approximations with fixed subspace size `m`.
  - :adaptive - uses adaptive Krylov approximations with internal timestepping.
- `m` - integer, default: `30`. Controls the size of Krylov subsapce if
  `krylov=:simple`, and the initial subspace size if `krylov=:adaptive`.
- `iop` - integer, default: `0`. If not zero, determines the length of the incomplete
  orthogonalization procedure (IOP) [^1]. Note that if the linear operator/jacobian is hermitian,
  then the Lanczos algorithm will always be used and the IOP setting is ignored.

#### Exponential Runge-Kutta Methods

These methods are all fixed timestepping only.

- `LawsonEuler` - First order exponential Euler scheme.
- `NorsettEuler` - First order exponential-RK scheme. Alias: `ETD1`.
- `ETD2` - Second order Exponential Time Differencing method (in development).
- `ETDRK2` - 2nd order exponential-RK scheme.
- `ETDRK3` - 3rd order exponential-RK scheme.
- `ETDRK4` - 4th order exponential-RK scheme.
- `HochOst4` - 4th order exponential-RK scheme with stiff order 4.

The methods are intended for semilinear problems constructed by
[`SplitODEProblem`](@ref split_ode_prob) or `SplitODEFunction`. They can
also be used for a general nonlinear problem, in which case the jacobian of the right
hand side is used as the linear operator in each time step.

Except for `ETD2`, all methods come with these options, which can be set in the methods'
constructor:

- `krylov` - boolean, default: `false`. Determines whether Krylov approximation or operator
  caching is used, the latter only available for semilinear problems.
- `m` - integer, default: `30`. Controls the size of Krylov subsapce.
- `iop` - integer, default: `0`. If not zero, determines the length of the incomplete
  orthogonalization procedure (IOP) [^1]. Note that if the linear operator/jacobian is hermitian,
  then the Lanczos algorithm will always be used and the IOP setting is ignored.
- `autodiff` and `chunksize`: autodiff control if problem is not semilinear and explicit jacobian
  is not given. See [Extra Options](#Extra-Options-1) for more details.

#### Adaptive Exponential Rosenbrock Methods

- `Exprb32` - 3rd order adaptive Exponential-Rosenbrock scheme.
- `Exprb43` - 4th order adaptive Exponential-Rosenbrock scheme.

The exponential rosenbrock methods cannot be applied to semilinear problems. Options for the
solvers are the same as [Exponential Runge-Kutta Methods](#Exponential-Runge-Kutta-Methods-1)
except that Krylov approximation is always used.

#### Exponential Propagation Iterative Runge-Kutta Methods (EPIRK)

These methods are all fixed timestepping only.

- `Exp4` - 4th order EPIRK scheme.
- `EPIRK4s3A` - 4th order EPIRK scheme with stiff order 4.
- `EPIRK4s3B` - 4th order EPIRK scheme with stiff order 4.
- `EPIRK5P1` - 5th order EPIRK scheme.
- `EPIRK5P2` - 5th order EPIRK scheme.
- `EPIRK5s3` - 5th order "horizontal" EPIRK scheme with stiff order 5. Broken.
- `EXPRB53s3`- 5th order EPIRK scheme with stiff order 5.

Options:

- `adaptive_krylov` - boolean, default: `true`. Determines if the adaptive Krylov algorithm
  with timestepping of Neisen & Wright is used.
- `m` - integer, default: `30`. Controls the size of Krylov subsapce, or the size for the
  first step if `adaptive_krylov=true`.
- `iop` - integer, default: `0`. If not zero, determines the length of the incomplete
  orthogonalization procedure (IOP) [^1]. Note that if the linear operator/jacobian is hermitian,
  then the Lanczos algorithm will always be used and the IOP setting is ignored.
- `autodiff` and `chunksize`: autodiff control if problem is not semilinear and explicit jacobian
  is not given. See [Extra Options](#Extra-Options-1) for more details.

It should be noted that many of the methods are still at an experimental stage of development,
and thus should be used with caution.

#### Multistep Methods

Quasi-constant stepping is the time stepping strategy which matches the classic
GEAR, LSODE,  and `ode15s` integrators. The variable-coefficient methods match
the ideas of the classic EPISODE integrator and early VODE designs. The Fixed
Leading Coefficient (FLC) methods match the behavior of the classic VODE and
Sundials CVODE integrator.

- `QNDF1` - An adaptive order 1 quasi-constant timestep L-stable numerical
  differentiation function (NDF) method. Optional parameter `kappa` defaults
  to Shampine's accuracy-optimal `-0.1850`.
- `QBDF1` - An adaptive order 1 L-stable BDF method. This is equivalent to
  implicit Euler but using the BDF error estimator.
- `ABDF2` - An adaptive order 2 L-stable fixed leading coefficient multistep
  BDF method.
- `QNDF2` - An adaptive order 2 quasi-constant timestep L-stable numerical
  differentiation function (NDF) method.
- `QBDF2` - An adaptive order 2 L-stable BDF method using quasi-constant timesteps.
- `QNDF` - An adaptive order quasi-constant timestep NDF method. Utilizes
  Shampine's accuracy-optimal `kappa` values as defaults (has a keyword argument
  for a tuple of `kappa` coefficients).
- `QBDF` - An adaptive order quasi-constant timestep BDF method.
- `JVODE_BDF` - An adaptive time adaptive order fixed-leading coefficient BDF
  method in Nordsieck form. In development.
- `MEBDF2` - The second order Modified Extended BDF method, which has improved
  stability properties over the standard BDF. Fixed timestep only.

#### Implicit Strong-Stability Preserving Runge-Kutta Methods for Hyperbolic PDEs (Conservation Laws)

- `SSPSDIRK2` - A second order A-L stable symplectic SDIRK method with the strong
  stability preserving (SSP) property (SSP coefficient 2). Fixed timestep only.

#### Extra Options

All of the Rosenbrock and SDIRK methods allow for specification of `linsolve`:
the linear solver which is used. For more information on specifying the linear
solver, see
[the manual page on solver specification](@ref linear_nonlinear).

Note that performance overload information (Jacobians etc.) are not used in this
mode. This can control autodifferentiation of the Jacobian as well.
For more information on specifying the nonlinear solver, see
[the manual page on solver specification](@ref linear_nonlinear).

Additionally, the Rosenbrock and SDIRK methods have differentiation
controls. In each of these, `autodiff` can be set to turn on/off
autodifferentiation, and `chunk_size` can be used to set the chunksize of the Dual
 numbers (see the
[documentation for ForwardDiff.jl for details](http://www.juliadiff.org/ForwardDiff.jl/stable/user/advanced/#Configuring-Chunk-Size-1)).
In addition, the Rosenbrock and SDIRK methods can set `diff_type`, which is the
type of numerical differentiation that is used (when autodifferentiation is
disabled). The choices are `Val{:central}`, `Val{:forward}` or `Val{:complex}`.

Examples:

```julia
sol = solve(prob,Rosenbrock23()) # Standard, uses autodiff
sol = solve(prob,Rosenbrock23(chunk_size=10)) # Autodiff with chunksize of 10
sol = solve(prob,Rosenbrock23(autodiff=false)) # Numerical differentiation with central differencing
sol = solve(prob,Rosenbrock23(autodiff=false,diff_type=Val{:forward})) # Numerical differentiation with forward differencing
```

#### Tableau Method

Additionally, there is the tableau method:

- `ExplicitRK` - A general Runge-Kutta solver which takes in a tableau. Can be adaptive. Tableaus
  are specified via the keyword argument `tab=tableau`. The default tableau is
  for Dormand-Prince 4/5. Other supplied tableaus can be found in the Supplied Tableaus section.

Example usage:

```julia
alg = ExplicitRK(tableau=constructDormandPrince())
solve(prob,alg)
```

#### CompositeAlgorithm

One unique feature of OrdinaryDiffEq.jl is the `CompositeAlgorithm`, which allows
you to, with very minimal overhead, design a multimethod which switches between
chosen algorithms as needed. The syntax is `CompositeAlgorithm(algtup,choice_function)`
where `algtup` is a tuple of OrdinaryDiffEq.jl algorithms, and `choice_function`
is a function which declares which method to use in the following step. For example,
we can design a multimethod which uses `Tsit5()` but switches to `Vern7()` whenever
`dt` is too small:

```julia
choice_function(integrator) = (Int(integrator.dt<0.001) + 1)
alg_switch = CompositeAlgorithm((Tsit5(),Vern7()),choice_function)
```

The `choice_function` takes in an `integrator` and thus all of the features
available in the [Integrator Interface](@ref integrator) can be used in the choice
function.

A helper algorithm was created for building 2-method automatic switching for
stiffness detection algorithms. This is the `AutoSwitch` algorithm with the
following options:

```julia
AutoSwitch(nonstiffalg::nAlg, stiffalg::sAlg;
           maxstiffstep=10, maxnonstiffstep=3,
           nonstifftol::T=9//10, stifftol::T=9//10,
           dtfac=2.0, stiffalgfirst=false)
```

The `nonstiffalg` must have an appropriate stiffness estimate built into the
method. The `stiffalg` can receive its estimate from the Jacobian calculation.
`maxstiffstep` is the number of stiffness detects before switching to the stiff
algorithm and `maxnonstiffstep` is vice versa. `nonstifftol` and `stifftol` are
the tolerances associated with the stiffness comparison against the stability
region. Decreasing `stifftol` makes switching to the non-stiff algorithm less
likely. Decreasing `nonstifftol` makes switching to the stiff algorithm more
likely. `dtfac` is the factor that `dt` is changed when switching: multiplied
when going from non-stiff to stiff and divided when going stiff to non-stiff.
`stiffalgfirst` denotes whether the first step should use the stiff algorithm.

#### Pre-Built Stiffness Detecting and Auto-Switching Algorithms

These methods require a `Autoalg(stiffalg)` to be chosen as the method to switch
to when the ODE is stiff. It can be any of the OrdinaryDiffEq.jl one-step stiff
methods and has all of the arguments of the `AutoSwitch` algorithm.

- `AutoTsit5` - `Tsit5` with automated switching.
- `AutoDP5` - `DP5` with automated switching.
- `AutoVern6` - `Vern6` with automated switching.
- `AutoVern7` - `Vern7` with automated switching.
- `AutoVern8` - `Vern8` with automated switching.
- `AutoVern9` - `Vern9` with automated switching.

Example:

```julia
tsidas_alg = AutoTsit5(Rodas5())
sol = solve(prob,tsidas_alg)

tsidas_alg = AutoTsit5(Rodas5(),nonstifftol = 11/10)
```

Is the `Tsit5` method with automatic switching to `Rodas5`.

### [Sundials.jl](@id ode_solve_sundials)

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use Sundials.jl:

```julia
]add Sundials
using Sundials
```

The Sundials suite is built around multistep methods. These methods are more efficient
than other methods when the cost of the function calculations is really high, but
for less costly functions the cost of nurturing the timestep overweighs the benefits.
However, the BDF method is a classic method for stiff equations and "generally works".

  - `CVODE_BDF` - CVode Backward Differentiation Formula (BDF) solver.
  - `CVODE_Adams` - CVode Adams-Moulton solver.
  - `ARKODE` - Explicit and ESDIRK Runge-Kutta methods of orders 2-8 depending
    on choice of options.

The Sundials algorithms all come with a 3rd order Hermite polynomial interpolation.
Note that the constructors for the Sundials algorithms take two main arguments:

  - `method` - This is the method for solving the implicit equation. For BDF this
    defaults to `:Newton` while for Adams this defaults to `:Functional`. These
    choices match the recommended pairing in the Sundials.jl manual. However,
    note that using the `:Newton` method may take less iterations but requires
    more memory than the `:Function` iteration approach.
  - `linear_solver` - This is the linear solver which is used in the `:Newton` method.

  The choices for the linear solver are:

  - `:Dense` - A dense linear solver.
  - `:Band` - A solver specialized for banded Jacobians. If used, you must set the
    position of the upper and lower non-zero diagonals via `jac_upper` and
    `jac_lower`.
  - `:LapackDense` - A version of the dense linear solver that uses the Julia-provided
    OpenBLAS-linked LAPACK for multithreaded operations. This will be faster than
    `:Dense` on larger systems but has noticable overhead on smaller (<100 ODE) systems.
  - `:LapackBand` - A version of the banded linear solver that uses the Julia-provided
    OpenBLAS-linked LAPACK for multithreaded operations. This will be faster than
    `:Band` on larger systems but has noticable overhead on smaller (<100 ODE) systems.
  - `:Diagonal` - This method is specialized for diagonal Jacobians.
  - `:GMRES` - A GMRES method. Recommended first choice Krylov method
  - `:BCG` - A Biconjugate gradient method.
  - `:PCG` - A preconditioned conjugate gradient method. Only for symmetric
    linear systems.
  - `:TFQMR` - A TFQMR method.
  - `:KLU` - A sparse factorization method. Requires that the user specifies a
    Jacobian. The Jacobian must be set as a sparse matrix in the `ODEProblem`
    type.

Example:

```julia
CVODE_BDF() # BDF method using Newton + Dense solver
CVODE_BDF(method=:Functional) # BDF method using Functional iterations
CVODE_BDF(linear_solver=:Band,jac_upper=3,jac_lower=3) # Banded solver with nonzero diagonals 3 up and 3 down
CVODE_BDF(linear_solver=:BCG) # Biconjugate gradient method                                   
```

The main options for `ARKODE` are the choice between explicit and implicit and
the method order, given via:

```julia
ARKODE(Sundials.Explicit()) # Solve with explicit tableau of default order 4
ARKODE(Sundials.Implicit(),order = 3) # Solve with explicit tableau of order 3
```

The order choices for explicit are 2 through 8 and for implicit 3 through 5.
Specific methods can also be set through the `etable` and `itable` options
for explicit and implicit tableaus respectively. The available tableaus are:

`etable`:

- `HEUN_EULER_2_1_2`: 2nd order Heun's method
- `BOGACKI_SHAMPINE_4_2_3`:
- `ARK324L2SA_ERK_4_2_3`: explicit portion of Kennedy and Carpenter's 3rd
  order method
- `ZONNEVELD_5_3_4`: 4th order explicit method
- `ARK436L2SA_ERK_6_3_4`: explicit portion of Kennedy and Carpenter's 4th
  order method
- `SAYFY_ABURUB_6_3_4`: 4th order explicit method
- `CASH_KARP_6_4_5`: 5th order explicit method
- `FEHLBERG_6_4_5`: Fehlberg's classic 5th order method
- `DORMAND_PRINCE_7_4_5`: the classic 5th order Dormand-Prince method
- `ARK548L2SA_ERK_8_4_5`: explicit portion of Kennedy and Carpenter's 5th
  order method
- `VERNER_8_5_6`: Verner's classic 5th order method
- `FEHLBERG_13_7_8`: Fehlberg's 8th order method

`itable`:

- `SDIRK_2_1_2`: An A-B-stable 2nd order SDIRK method
- `BILLINGTON_3_3_2`: A second order method with a 3rd order error predictor
  of less stability
- `TRBDF2_3_3_2`: The classic TR-BDF2 method
- `KVAERNO_4_2_3`: an L-stable 3rd order ESDIRK method
- `ARK324L2SA_DIRK_4_2_3`: implicit portion of Kennedy and Carpenter's 3th
  order method
- `CASH_5_2_4`: Cash's 4th order L-stable SDIRK method
- `CASH_5_3_4`: Cash's 2nd 4th order L-stable SDIRK method
- `SDIRK_5_3_4`: Hairer's 4th order SDIRK method
- `KVAERNO_5_3_4`: Kvaerno's 4th order ESDIRK method
- `ARK436L2SA_DIRK_6_3_4`: implicit portion of Kennedy and Carpenter's 4th
  order method
- `KVAERNO_7_4_5`: Kvaerno's 5th order ESDIRK method
- `ARK548L2SA_DIRK_8_4_5`: implicit portion of Kennedy and Carpenter's 5th
  order method

These can be set for example via:

```julia
ARKODE(Sundials.Explicit(),etable = Sundials.DORMAND_PRINCE_7_4_5)
ARKODE(Sundials.Implicit(),itable = Sundials.KVAERNO_4_2_3)
```

All of the additional options are available. The full constructor is:

```julia
CVODE_BDF(;method=:Newton,linear_solver=:Dense,
          jac_upper=0,jac_lower=0,
          stored_upper = jac_upper + jac_lower,
          non_zero=0,krylov_dim=0,
          stability_limit_detect=false,
          max_hnil_warns = 10,
          max_order = 5,
          max_error_test_failures = 7,
          max_nonlinear_iters = 3,
          max_convergence_failures = 10,
          prec = nothing, prec_side = 0)

CVODE_Adams(;method=:Functional,linear_solver=:None,
            jac_upper=0,jac_lower=0,
            stored_upper = jac_upper + jac_lower,
            krylov_dim=0,
            stability_limit_detect=false,
            max_hnil_warns = 10,
            max_order = 12,
            max_error_test_failures = 7,
            max_nonlinear_iters = 3,
            max_convergence_failures = 10,
            prec = nothing, psetup = nothing, prec_side = 0)

ARKODE(stiffness=Sundials.Implicit();
      method=:Newton,linear_solver=:Dense,
      jac_upper=0,jac_lower=0,stored_upper = jac_upper+jac_lower,
      non_zero=0,krylov_dim=0,
      max_hnil_warns = 10,
      max_error_test_failures = 7,
      max_nonlinear_iters = 3,
      max_convergence_failures = 10,
      predictor_method = 0,
      nonlinear_convergence_coefficient = 0.1,
      dense_order = 3,
      order = 4,
      set_optimal_params = false,
      crdown = 0.3,
      dgmax = 0.2,
      rdiv = 2.3,
      msbp = 20,
      adaptivity_method = 0,
      prec = nothing, psetup = nothing, prec_side = 0
      )
```

See [the CVODE manual](https://computation.llnl.gov/sites/default/files/public/cv_guide.pdf)
and the [ARKODE manual](https://computation.llnl.gov/sites/default/files/public/ark_guide.pdf)
for details on the additional options.

Note that here `prec` is a preconditioner function
`prec(z,r,p,t,y,fy,gamma,delta,lr)` where:

- `z`: the computed output vector
- `r`: the right-hand side vector of the linear system
- `p`: the parameters
- `t`: the current independent variable
- `du`: the current value of `f(u,p,t)`
- `gamma`: the `gamma` of `W = M - gamma*J`
- `delta`: the iterative method tolerance
- `lr`: a flag for whether `lr=1` (left) or `lr=2` (right)
  preconditioning

and `psetup` is the preconditioner setup function for pre-computing Jacobian
information. Where:

- `p`: the parameters
- `t`: the current independent variable
- `u`: the current state
- `du`: the current `f(u,p,t)`
- `jok`: a bool indicating whether the Jacobian needs to be updated
- `jcurPtr`: a reference to an Int for whether the Jacobian was updated.
  `jcurPtr[]=true` should be set if the Jacobian was updated, and
  `jcurPtr[]=false` should be set if the Jacobian was not updated.
- `gamma`: the `gamma` of `W = M - gamma*J`

`psetup` is optional when `prec` is set.

### ODEInterface.jl

The ODEInterface algorithms are the classic Fortran algorithms. While the
non-stiff algorithms are superseded by the more featured and higher performance
Julia implementations from OrdinaryDiffEq.jl, the stiff solvers such as `radau`
are some of the most efficient methods available (but are restricted for use on
arrays of Float64).

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use ODEInterfaceDiffEq.jl:

```julia
]add ODEInterfaceDiffEq
using ODEInterfaceDiffEq
```

  - `dopri5` - Hairer's classic implementation of the Dormand-Prince 4/5 method.
  - `dop853` - Explicit Runge-Kutta 8(5,3) by Dormand-Prince.
  - `odex` - GBS extrapolation-algorithm based on the midpoint rule.
  - `seulex` - Extrapolation-algorithm based on the linear implicit Euler method.
  - `radau` - Implicit Runge-Kutta (Radau IIA) of variable order between 5 and 13.
  - `radau5` - Implicit Runge-Kutta method (Radau IIA) of order 5.
  - `rodas` - Rosenbrock 4(3) method.
  - `ddeabm` - Adams-Bashforth-Moulton Predictor-Corrector method (order between
    1 and 12)
  - `ddebdf` - Backward Differentiation Formula (orders between 1 and 5)

Note that while the output only has a linear interpolation, a higher order
interpolation is used for intermediate dense output for `saveat` and for
event handling.

### LSODA.jl

This setup provides a wrapper to the algorithm LSODA, a well-known method which uses switching
to solve both stiff and non-stiff equations.

  - `lsoda` - The LSODA wrapper algorithm.

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use LSODA.jl:

```julia
]add LSODA
using LSODA
```

### SimpleDiffEq.jl

This setup provides access to simplified versions of a few ODE solvers. They
mostly exist for experimentation, but offer shorter compile times. They have
limitations compared to OrdinaryDiffEq.jl and are not generally faster.

  - `SimpleTsit5` - A fixed timestep integrator form of Tsit5. Not compatible
    with events.
  - `SimpleATsit5` - An adaptive Tsit5 with an interpolation in its simplest
    form. Not compatible with events.
  - `GPUSimpleATsit5` - A version of `SimpleATsit5` without the integrator
    interface. Only allows `solve`.

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use SimpleDiffEq.jl:

```julia
]add SimpleDiffEq
using SimpleDiffEq
```

### ODE.jl


Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use ODE.jl:

```julia
]add ODE
using ODE
```

  - `ode23` - Bogacki-Shampine's order 2/3 Runge-Kutta  method
  - `ode45` - A Dormand-Prince order 4/5 Runge-Kutta method
  - `ode23s` - A modified Rosenbrock order 2/3 method due to Shampine
  - `ode78` - A Fehlburg order 7/8 Runge-Kutta method
  - `ode4` - The classic Runge-Kutta order 4 method
  - `ode4ms` - A fixed-step, fixed order Adams-Bashforth-Moulton method†
  - `ode4s` - A 4th order Rosenbrock method due to Shampine


†: Does not step to the interval endpoint. This can cause issues with discontinuity
detection, and [discrete variables need to be updated appropriately](@ref diffeq_arrays).

### MATLABDiffEq.jl

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use MATLABDiffEq.jl:

```julia
]add https://github.com/JuliaDiffEq/MATLABDiffEq.jl
using MATLABDiffEq
```

This requires a licensed MATLAB installation. The available methods are:

  - `MATLABDiffEq.ode23`
  - `MATLABDiffEq.ode45`
  - `MATLABDiffEq.ode113`
  - `MATLABDiffEq.ode23s`
  - `MATLABDiffEq.ode23t`
  - `MATLABDiffEq.ode23tb`
  - `MATLABDiffEq.ode15s`
  - `MATLABDiffEq.ode15i`

For more information on these algorithms, see
[the MATLAB documentation](https://www.mathworks.com/help/matlab/math/choose-an-ode-solver.html).

### SciPyDiffEq.jl

[SciPyDiffEq.jl](https://github.com/JuliaDiffEq/SciPyDiffEq.jl) is a wrapper over SciPy for
easing the transition of new users (same exact results!)
and benchmarking. This wrapper uses Julia's JIT acceleration to accelerate about 3x over SciPy+Numba,
but it is still around 1000x slower than the pure-Julia methods and thus should probably be used
sparingly.

Note that this setup is not automatically included
with DifferentialEquations.jl. To use the following algorithms, you must install
and use SciPyDiffEq.jl:

```julia
]add https://github.com/JuliaDiffEq/SciPyDiffEq.jl
using SciPyDiffEq
```

The available methods are:

- `SciPyDiffEq.RK45`
- `SciPyDiffEq.RK23`
- `SciPyDiffEq.Radau`
- `SciPyDiffEq.BDF`
- `SciPyDiffEq.LSODA`

### deSolveDiffEq.jl

[deSolveDiffEq.jl](https://github.com/JuliaDiffEq/deSolveDiffEq.jl) is a wrapper over R's deSolve for
easing the transition of new users (same exact results!)
and benchmarking. This wrapper is around 1000x slower than the pure-Julia methods (~2x-3x overhead
from directly using R) and thus should probably be used sparingly.

Note that this setup is not automatically included
with DifferentialEquations.jl. To use the following algorithms, you must install
and use deSolveDiffEq.jl:

```julia
]add https://github.com/JuliaDiffEq/deSolveDiffEq.jl
using deSolveDiffEq
```

The available methods are:

- `deSolveDiffEq.lsoda`
- `deSolveDiffEq.lsode`
- `deSolveDiffEq.lsodes`
- `deSolveDiffEq.lsodar`
- `deSolveDiffEq.vode`
- `deSolveDiffEq.daspk`
- `deSolveDiffEq.euler`
- `deSolveDiffEq.rk4`
- `deSolveDiffEq.ode23`
- `deSolveDiffEq.ode45`
- `deSolveDiffEq.radau`
- `deSolveDiffEq.bdf`
- `deSolveDiffEq.bdf_d`
- `deSolveDiffEq.adams`
- `deSolveDiffEq.impAdams`
- `deSolveDiffEq.impAdams_d`

### GeometricIntegrators.jl

GeometricIntegrators.jl is a set of fixed timestep algorithms written in Julia.
Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and use
GeometricIntegratorsDiffEq.jl:

```julia
]add https://github.com/JuliaDiffEq/GeometricIntegratorsDiffEq.jl
using GeometricIntegratorsDiffEq
```

- `GIEuler` - 1st order Euler method
- `GIMidpoint` - 2nd order explicit midpoint method
- `GIHeun` - 2nd order Heun's method
- `GIKutta` - 3rd order Kutta's method
- `GIERK4` - standard 4th order Runge-Kutta
- `GIERK438` - 4th order Runge-Kutta, 3/8's rule
- `GIImplicitEuler` - 1st order implicit Euler method
- `GIImplicitMidpoint` - 2nd order implicit midpoint method
- `GIRadIIA2` - 2-stage order 3 Radau-IIA
- `GIRadIIA3` - 3-stage order 5 Radau-IIA
- `GISRK3` - 3-stage order 4 symmetric Runge-Kutta method
- `GIGLRK(s)` - Gauss-Legendre Runge-Kutta method of order 2s

Note that all of these methods require the user supplies `dt`.

### BridgeDiffEq.jl

Bridge.jl is a set of fixed timestep algorithms written in Julia. These methods
are made and optimized for out-of-place functions on immutable (static vector)
types. Note that this setup is not automatically included with
DifferentialEquations.jl. To use the following algorithms, you must install and
use BridgeDiffEq.jl:

```julia
]add https://github.com/JuliaDiffEq/BridgeDiffEq.jl
using BridgeDiffEq
```

- `BridgeR3` - 3rd order Ralston method
- `BridgeBS3` - 3rd order Bogacki-Shampine method

### TaylorIntegration.jl

TaylorIntegration.jl is a pure-Julia implementation of an adaptive order Taylor
series method for high accuracy integration of ODEs. These methods are optimized
when the absolute tolerance is required to be very low.
Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and
use TaylorIntegration.jl:

```julia
]add TaylorIntegration
using TaylorIntegration
```

- `TaylorMethod(order)` - Taylor integration method with maximal `order` (required)

Note: this method is much faster if you put `@taylorize` on your derivative function!

### QuDiffEq.jl

QuDiffEq.jl is a pacakge for solving differential equations using quantum algorithm. It makes use of the Yao framework for simulating quantum circuits.

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and
use QuDiffEq.jl:

```julia
]add https://github.com/QuantumBFS/QuDiffEq.jl
using QuDiffEq
```

- `QuLDE(k)` - Algorithm based on truncated Taylor series. The method linearizes a system of non-linear differential equations and solves the resultant by means of a quantum circuit. `k` selects the order in the Taylor series aprroximation (for the quantum circuit).
- `QuNLDE(k,ϵ)`- Algorithm uses forward Euler to solve quadratc differential equations. `k` selects the order in the Taylor series aprroximation (for the quantum circuit). `ϵ` sets the precision for Hamiltonian evolution.

### NeuralNetDiffEq.jl

This method trains a neural network using Flux.jl to approximate the solution of the
ODE. Currently this method isn't competitive but it is a fun curiosity that will be
improved with future integration with Zygote.

Note that this setup is not automatically included with DifferentialEquations.jl.
To use the following algorithms, you must install and
use NeuralNetDiffEq.jl:

```julia
]add NeuralNetDiffEq
using NeuralNetDiffEq
```

- `nnode(chain,opt=ADAM(0.1))` - Defines a neural network solver which utilizes a Flux.jl
  `chain` under the hood which must be supplied by the user. Defaults to using the ADAM
  optimization method, but the user can pass any Flux.jl optimizer.

### List of Supplied Tableaus

A large variety of tableaus have been supplied by default, via DiffEqDevTools.jl.
The list of tableaus can be found in [the developer docs](https://devdocs.juliadiffeq.org/dev/internals/tableaus/).
To use them, note you must install the library:

```julia
]add DiffEqDevTools
using DiffEqDevTools
```

For the most useful and common algorithms, a hand-optimized version is supplied
in OrdinaryDiffEq.jl which is recommended for general uses (i.e. use `DP5`
instead of `ExplicitRK` with `tableau=constructDormandPrince()`). However, these
serve as a good method for comparing between tableaus and understanding the
pros/cons of the methods. Implemented are every published tableau (that I know
exists). Note that user-defined tableaus also are accepted. To see how to define
a tableau, checkout the [premade tableau source code](https://github.com/JuliaDiffEq/DiffEqDevTools.jl/blob/master/src/ode_tableaus.jl).
Tableau docstrings should have appropriate citations (if not, file an issue).

Plot recipes are provided which will plot the stability region for a given tableau.

[^1]: Koskela, A. (2015). Approximating the matrix exponential of an advection-diffusion operator using the incomplete orthogonalization method. In Numerical Mathematics and Advanced Applications-ENUMATH 2013 (pp. 345-353). Springer, Cham.
