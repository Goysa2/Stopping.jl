# Stopping

## Purpose

Tools to ease the uniformization of stopping criteria in iterative solvers.

When a solver is called on an optimization model, four outcome may happen:

1. the approximate solution is obtained, the problem is considered solved
2. the problem is declared unbounded
3. the maximum available ressources is not sufficient to compute the solution
4. some algorithm dependent failure happens

This tool eases the first 3 items above. It defines a type

    type Stopping
        problem       :: Any          # an arbitrary instance of a problem
        meta          :: StoppingMeta # contains the used parameters
        current_state :: State        # the current state

The StoppingMeta provides default tolerances, maximum ressources, ...  

We provide some specialization of the GenericStopping for instance :
  * for non-linear programming (NLPStopping);
  * or for 1d optimization (LineSearchStopping).
In these examples, the function `optimality_residual` computes the residual of the optimality conditions is an additional attribute of the type.

## Functions

The tool provides two functions:
* `start!(nlp,s,x0)` initializes the time counter and the tolerance at the starting point. This function is called once at the beginning of an algorithm.
* `stop(nlp,s, iter, x, gradf)` verifies if the tolerance is reached for `x` or if the maximum ressources is reached. This function returns booleans optimal, unbounded, tired; moreover, it returns the elapsed time, and fine grain information. Usually, only the four first outputs are used. This function is called at every iteration and, complemented with algorithm specific conditions, is the stopping criterion.
* start!, stop!, update_and_start!, update_and_stop!

## How to install

The stopping package can be installed and tested through the Julia package manager:

```julia
(v0.7) pkg> add https://github.com/Goysa2/Stopping.jl
(v0.7) pkg> test Stopping
```
Note that the package [State.jl](https://github.com/Goysa2/State.jl) is required and can be installed also through the package manager:
```julia
(v0.7) pkg> add https://github.com/Goysa2/State.jl
```
## Example

As an example, a naïve version of the Newton method is provided. First we import the packages:
```
using NLPModels, State, Stopping
```

We create an uncontrained quadratic optimization problem using [NLPModels](https://github.com/JuliaSmoothOptimizers/NLPModels.jl):
```
A = rand(5, 5); Q = A' * A;

f(x) = 0.5 * x' * Q * x
nlp = ADNLPModel(f,  ones(5))
```

We use our NLPStopping structure by creating our State and Stopping:

```
nlp_at_x = NLPAtX(ones(5))
stop_nlp = NLPStopping(nlp, (x,y) -> Stopping.unconstrained(x,y), nlp_at_x)
```

Now a basic version of Newton to illustrate how to use State and Stopping.

```
function newton(stp :: NLPStopping)
    state = stp.current_state; xt = state.x;
    update!(state, x = xt, gx = grad(stp.pb, xt), Hx = hess(stp.pb, xt))
    OK = start!(stp)

    while !OK
        d = -inv(state.Hx) * state.gx

        xt = xt + d

        update!(state, x = xt, gx = grad(stp.pb, xt), Hx = hess(stp.pb, xt))

        OK = stop!(stp)
    end

    return stp
end

stop_nlp = newton(stop_nlp)
```

We can look at the meta to know what happened
```
@show stop_nlp.meta.tired #ans: false
@show stop_nlp.meta.unbounded #ans: false
@show stop_nlp.meta.optimal #ans: true
```

We reached optimality!

## Long-Term Goals

Future work will address constrained problems. Then, fine grained information will consists in the number of constraint, their gradient etc. evaluation. The optimality conditions will be based on KKT equations. Separate tolerances for optimality and feasibility will be developed.
