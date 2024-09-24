
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

julia> rng = StableRNG(1729)
StableRNGs.LehmerRNG(state=0x00000000000000000000000000000d83)

julia> stochasticmodel(sirrates, rng, u0, 1:10, sirtransitionmatrix)
10×3 Matrix{Int64}:
 9  1  0
 7  3  0
 5  5  0
 3  4  3
 2  4  4
 1  4  5
 1  4  5
 1  4  5
 0  5  5
 0  5  5
```

Note that the argument `rng` is optional in all functions. It is provided here along with use of `StableRNGs.jl` to ensure reproducability of the examples.

The frequency of recording results can be changed with the `saveat` keyword argument, or by specifying a frequency in the timerange.

```jldoctest label
julia> stochasticmodel(sirrates, rng, u0, ( 1, 10 ), sirtransitionmatrix; saveat=2)
5×3 Matrix{Int64}:
 9  1  0
 1  9  0
 0  6  4
 0  4  6
 0  1  9

julia> stochasticmodel(sirrates, rng, u0, 1:2:10, sirtransitionmatrix)
5×3 Matrix{Int64}:
 9  1  0
 0  8  2
 0  6  4
 0  4  6
 0  1  9
```

We can also save the result at every transition. In this case, a vector of transition times is also returned.

```jldoctest label
julia> outputs, tvalues = stochasticmodel(sirrates, rng, u0, 10, sirtransitionmatrix; saveall=true)        
(outputs = [9 1 0; 8 2 0; … ; 0 0 10; 0 0 10], outputtimes = [1.0, 1.1937416718324927, 1.3152122995740403, 1.3914161806081558, 1.6662787047527419, 2.184717032201454, 2.6185520422547204, 2.7345941562589804, 3.189110023787074, 3.5695945359298555  …  3.8596475013976566, 4.545515561119796, 4.864699439234508, 5.278104163204452, 5.3511369134396265, 5.613730552879197, 6.549409122918718, 6.6204713368987695, 9.503801051031607, 10.0])

julia> outputs
21×3 Matrix{Int64}:
 9  1   0
 8  2   0
 7  3   0
 6  4   0
 6  3   1
 5  4   1
 4  5   1
 3  6   1
 2  7   1
 1  8   1
 ⋮
 0  7   3
 0  6   4
 0  5   5
 0  4   6
 0  3   7
 0  2   8
 0  1   9
 0  0  10
 0  0  10

julia> tvalues
21-element Vector{Float64}:
  1.0
  1.1937416718324927
  1.3152122995740403
  1.3914161806081558
  1.6662787047527419
  2.184717032201454
  2.6185520422547204
  2.7345941562589804
  3.189110023787074
  3.5695945359298555
  ⋮
  4.545515561119796
  4.864699439234508
  5.278104163204452
  5.3511369134396265
  5.613730552879197
  6.549409122918718
  6.6204713368987695
  9.503801051031607
 10.0
```

The default setting is to assume that the function calculating transition rates accepts the time. This can be cancelled by setting `broadcast_t=false` 

```jldoctest label
julia> function sirrates_2(u)
           s, i, r = u
           n = s + i + r
           return [
               2 * s * i / n,  # infection rate
               0.2 * i  # recovery rate
           ]
       end
sirrates_2 (generic function with 1 method)

julia> stochasticmodel(sirrates_2, rng, u0, 1:10, sirtransitionmatrix; broadcast_t=false)
10×3 Matrix{Int64}:
 9  1  0
 4  6  0
 0  9  1
 0  6  4
 0  6  4
 0  5  5
 0  5  5
 0  5  5
 0  4  6
 0  2  8
```

A set of parameters can also be sent to the function. This can be a vector or any other structure. 

```jldoctest label
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

julia> stochasticmodel(sirrates_3, rng, u0, 1:10, p, sirtransitionmatrix)
10×3 Matrix{Int64}:
 9  1  0
 6  4  0
 5  3  2
 5  2  3
 4  2  4
 3  2  5
 2  3  5
 2  1  7
 1  2  7
 1  1  8

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

julia> stochasticmodel(sirrates_4, rng, u0, 1:10, p2, sirtransitionmatrix)
10×3 Matrix{Int64}:
 9  1  0
 6  3  1
 3  6  1
 0  5  5
 0  5  5
 0  5  5
 0  5  5
 0  5  5
 0  4  6
 0  3  7
```
