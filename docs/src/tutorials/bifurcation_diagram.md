# Bifurcation Diagrams
Bifurcation diagrams can be produced from Catalyst generated models through the use of the [BifurcationKit.jl](https://github.com/rveltz/BifurcationKit.jl/) package. This tutorial gives a simple example of how to create such a bifurcation diagram. *Please note,* this tutorial is written for BifurcationKit version 0.1.11 and has not yet been updated to work on newer versions like 0.2 and up.

First, we declare our model. For our example we will use a bistable switch, but which also contains a Hopf bifurcation.
```julia
using Catalyst
rn = @reaction_network begin
    (v0+v*(S*X)^n/((S*X)^n+(D*A)^n+K^n),d), ∅ ↔ X
    (τ*X,τ), ∅ ↔ A
end S D τ v0 v K n d
```
Next, BifurcationKit requires another form for the system function and Jacobian than what is used by the SciML ecosystem, so we need to declare these:
```julia
odefun = ODEFunction(convert(ODESystem,rn),jac=true)
F = (u,p) -> odefun(u,p,0)      
J = (u,p) -> odefun.jac(u,p,0)
```
We also need to specify the system parameters for which we wish to plot the bifurcation diagram:
```julia
params = [1.,9.,0.001,0.01,2.,20.,3,0.05]
```
Finally, we also need to set the input required to make the diagram. This is the index (in the parameter array) of the bifurcation parameter, the range over which we wish to plot the bifurcation diagram, as well as for which variable we wish to plot the steady state values in the diagram.
```julia
p_idx = 1            # The index of the bifurcation parameter.
p_span = (0.1,20.)   # The parameter range for the bifurcation diagram.
plot_var_idx = 1     # The index of the variable we plot in the bifurcation diagram.
```

Now we need to fetch the required packages to create the bifurcation diagram.
```julia
using BifurcationKit, Plots, LinearAlgebra, Setfield
```
Next, we need to specify the input options for the pseudo-arclength continuation method which produces the diagram. We will use a [deflated continuation](https://rveltz.github.io/BifurcationKit.jl/dev/DeflatedContinuation/).
```julia
opts = ContinuationPar( dsmax = 0.05,        # Maximum arclength value of the pseudo-arc length continuation method.
                        dsmin = 1e-4,        # Minimum arclength value of the pseudo-arc length continuation method.
                        ds=0.001,            # Initial arclength value of the pseudo-arc length continuation method (should be positive).
                        maxSteps = 100000,   # The maximum number of steps.
                        pMin = p_span[1],    # Minimum p-vale (if hit, the method stops).
                        pMax = p_span[2],    # Maximum p-vale (if hit, the method stops).
                        detectBifurcation=3, # Value in {0,1,2,3} determening to what extent bofurcation points are detected (0 means nothing is done, 3 both them and there localisation are detected).
                        newtonOptions = NewtonPar(tol = 1e-9, verbose = false, maxIter = 15)) #Parameters to the newton solver (when finding fixed points) see BifurcationKit documentation.
                        
DO = DeflationOperator( 2,      # Algorithm parameter required when using deflated continuation, see BifurcationKit documentation.
                        dot,    # Algorithm parameter required when using deflated continuation, see BifurcationKit documentation.
                        1.,     # Algorithm parameter required when using deflated continuation, see BifurcationKit documentation.
                        [fill(0.,length(rn.states))]); # Guess(es) of the fixed point for the initial parameter set. Do not need to be exact.

```
With all this done, we can compute the bifurcations:
```julia
params_input = setindex!(copy(params),p_span[1],p_idx)                                # The input parameter values have to start at the first index of our parameter span.
branches, = continuation(F, J, params_input, (@lens _[p_idx]), opts , DO,             # Gives our input.
    verbosity = 0, plot=false,                                                    # We do not want to display, or plot, intermediary results.
    recordFromSolution = (x, p) -> x[plot_var_idx],                                   # How we wish to print the output in the diagram. Here we simply want the value of the target varriable.
    perturbSolution = (x,p,id) -> (x  .+ 0.8 .* rand(length(x))),                     # Parameter for the continuation method, see BifurcationKit documentation.
    callbackN = BifurcationKit.cbMaxNorm(1e7))                      # Parameter for the continuation method, see BifurcationKit documentation.
```
which can then be plotted using
```julia
plot(branches...,xlabel=rn.ps[1],ylabel=Symbol(rn.states[1].val.f),markersize=4,
     ylim=(0.,Inf),                                  # This ensures we do not display negative solutions.
     color=:blue,                                    # Otherwise each individual branch will have their separate colors.
     plotbifpoints = false, putbifptlegend = false,  # Plots the bifurcation point(s).
     linewidthstable=4, linewidthunstable=1)         # Stable/unstable values are distinguised by line thickness.
```
![bifurcation_diagram1](../assets/bifurcation_diagram1.svg)

Here the Hopf bifurcation is amrked with a blue square. The region with a thiner linewidth corresponds to unstable steady states. If one wishes to mark these differently it is possible to plot the individual brances separatly:
```julia
plot(branches[1],lw=4,color=map(i->(i==0) ? :blue : :red, getproperty.(branches[1].branch,:n_unstable)))
plot!(branches[3],lw=4,color=map(i->(i==0) ? :blue : :red, getproperty.(branches[3].branch,:n_unstable)))
plot!(branches[4],lw=4,color=map(i->(i==0) ? :blue : :red, getproperty.(branches[4].branch,:n_unstable)),plotbifpoints = false,xlabel=rn.ps[1],ylabel=Symbol(rn.states[1].val.f))
```
![bifurcation_diagram2](../assets/bifurcation_diagram2.svg)

(Note that the second branch corresponds to a negative steady state, which is biological irrelevant, and we hence do not plot)
