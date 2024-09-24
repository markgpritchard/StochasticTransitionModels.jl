
```@meta
DocTestSetup = quote
    using StochasticTransitionModels
    using StableRNGs 
end
```

# StochasticTransitionModels.jl

This package is mainly built for my own use. It allows generation of a stochastic continuous-time compartmental model.

```@docs
stochastictransitionmodels
```

## Examples of usage 

### Susceptible--infectious--recovered model 

To generate a stochastic susceptible--infectious--recovered model and record its output daily for ten days, we first create a function that gives the rate of each possible transition (infection or recovery)

```jldoctest label
julia> using StochasticTransitionModels

julia> function sirrates(u, t)
           s, i, r = u
           n = s + i + r
           return [
               2 * s * i / n,  # infection rate
               0.2 * i  # recovery rate
           ]
       end
sirrates (generic function with 1 method)
```

We create a vector of intitial conditions, `u0`

```jldoctest label
julia> u0 = [ 9, 1, 0 ]
3-element Vector{Int64}:
 9
 1
 0
```

We then need a matrix that shows how each compartment changes size for each possible transition. Each column must be a compartment, in the same order as `u0`, and each row must be a transition in the same order as provided in the accompanying function 

```jldoctest label
julia> sirtransitionmatrix = [
           # s   i   r
            -1   1   0   # infection
             0  -1   1   # recovery
             ]
2×3 Matrix{Int64}:
 -1   1  0
  0  -1  1
```

The default settings record outputs at each integer time point 

```jldoctest label
julia> using Random, StableRNGs

julia> rng = StableRNG(1)
StableRNGs.LehmerRNG(state=0x00000000000000000000000000000003)

julia> Random.seed!(1729)
TaskLocalRNG()

julia> stochasticmodel(sirrates, u0, 1:10, sirtransitionmatrix)
10×3 Matrix{Int64}:
 9  1  0
 9  0  1
 9  0  1
 9  0  1
 9  0  1
 9  0  1
 9  0  1
 9  0  1
 9  0  1
 9  0  1
```

The frequency of recording results can be changed with the `saveat` keyword argument, or by specifying a frequency in the timerange.

```jldoctest label
julia> Random.seed!(1730)
TaskLocalRNG()

julia> stochasticmodel(sirrates, u0, ( 1, 10 ), sirtransitionmatrix; saveat=2)
5×3 Matrix{Int64}:
 9  1  0
 1  8  1
 0  6  4
 0  3  7
 0  2  8

julia> stochasticmodel(sirrates, u0, 1:2:10, sirtransitionmatrix)
5×3 Matrix{Int64}:
 9  1  0
 9  0  1
 9  0  1
 9  0  1
 9  0  1
```

We can also save the result at every transition. In this case, a vector of transition times is also returned.

```jldoctest label
julia> Random.seed!(1731)
TaskLocalRNG()

julia> outputs, tvalues = stochasticmodel(sirrates, u0, 10, sirtransitionmatrix; saveall=true)        
(outputs = [9 1 0; 8 2 0; … ; 0 4 6; 0 4 6], outputtimes = [1.0, 1.0761929104852557, 1.5375630069338873, 1.5910512363731852, 1.636983208802689, 1.7892290640569253, 1.8774277795661412, 2.526637658536853, 3.0781466604969685, 3.451772699456614, 4.009563749103505, 4.105144448718718, 4.544957800798036, 5.276317670230953, 5.692753913995566, 7.609066315338376, 10.0])

julia> outputs
17×3 Matrix{Int64}:
 9  1  0
 8  2  0
 7  3  0
 6  4  0
 6  3  1
 5  4  1
 4  5  1
 3  6  1
 2  7  1
 1  8  1
 0  9  1
 0  8  2
 0  7  3
 0  6  4
 0  5  5
 0  4  6
 0  4  6

julia> tvalues
17-element Vector{Float64}:
  1.0
  1.0761929104852557
  1.5375630069338873
  1.5910512363731852
  1.636983208802689
  1.7892290640569253
  1.8774277795661412
  2.526637658536853
  3.0781466604969685
  3.451772699456614
  4.009563749103505
  4.105144448718718
  4.544957800798036
  5.276317670230953
  5.692753913995566
  7.609066315338376
 10.0
```

The default setting is to assume that the function calculating transition rates accepts the time. This can be cancelled by setting `broadcast_t=false` 

```jldoctest label
julia> Random.seed!(1732)
TaskLocalRNG()

julia> function sirrates_2(u)
           s, i, r = u
           n = s + i + r
           return [
               2 * s * i / n,  # infection rate
               0.2 * i  # recovery rate
           ]
       end
sirrates_2 (generic function with 1 method)

julia> stochasticmodel(sirrates_2, u0, 1:10, sirtransitionmatrix; broadcast_t=false)
10×3 Matrix{Int64}:
 9  1  0
 8  2  0
 3  7  0
 1  7  2
 0  6  4
 0  4  6
 0  4  6
 0  3  7
 0  3  7
 0  3  7
```

A set of parameters can also be sent to the function. This can be a vector or any other structure. 

```jldoctest label
julia> Random.seed!(1733)
TaskLocalRNG()

julia> function sirrates_3(u, t, p)
           s, i, r = u
           n = s + i + r
           return [
               p[1] * s * i / n,  # infection rate
               p[2] * i  # recovery rate
           ]
       end
sirrates_3 (generic function with 1 method)

julia> p = [ 2, 0.2 ]
2-element Vector{Float64}:
 2.0
 0.2

julia> stochasticmodel(sirrates_3, u0, 1:10, p, sirtransitionmatrix)
10×3 Matrix{Int64}:
 9  1  0
 9  1  0
 4  6  0
 1  8  1
 0  7  3
 0  7  3
 0  6  4
 0  5  5
 0  4  6
 0  4  6

julia> function sirrates_4(u, t, p)
           s, i, r = u
           n = s + i + r
           return [
               p.β * s * i / n,  # infection rate
               p.γ * i  # recovery rate
           ]
       end
sirrates_4 (generic function with 1 method)

julia> struct Parameters
           β   :: Float64
           γ   :: Float64
       end

julia> p2 = Parameters(2.0, 0.2)
Parameters(2.0, 0.2)

julia> stochasticmodel(sirrates_4, u0, 1:10, p2, sirtransitionmatrix)
10×3 Matrix{Int64}:
 9  1  0
 8  2  0
 5  3  2
 1  7  2
 0  5  5
 0  3  7
 0  3  7
 0  3  7
 0  3  7
 0  2  8
```
