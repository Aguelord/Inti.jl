# Getting started

!!! note "Important points covered in this tutorial"
      - Create a domain and its accompanying mesh
      - Solve a basic boundary integral equation
      - Visualize the solution

This first tutorial will guide you through the basic steps of setting up a
boundary integral equation problem and solving it using Inti.jl. We will
consider the classic Helmholtz scattering problem in 2D, and solve it using a
*direct* boundary integral formulation. More precisely, letting ``\Omega \subset
\mathbb{R}^d`` be a bounded domain, and denoting by ``\Gamma = \partial \Omega``
its boundary, we will solve the following Helmholtz problem:

```math
\begin{aligned}
    \Delta u + k^2 u  &= 0 \quad &&\text{in} \quad \mathbb{R}^d \setminus \bar{\Omega},\\
    \partial_\nu u &= g \quad &&\text{on} \quad \Gamma,\\
    \sqrt{r} \left( \frac{\partial u}{\partial r} - i k u \right) &= o(1) \quad &&\text{as} \quad r \to \infty,
\end{aligned}
```

where ``g`` is a (given) boundary datum, ``\nu`` is the outward unit normal to
``\Gamma``, ``k`` is the constant wavenumber, and ``r = ||\boldsymbol{x}||`` is
the radial coordinate. 

!!! note "Sommerfeld radiation condition"
    The last condition is the *Sommerfeld radiation condition, and is required
    to ensure the uniqueness of the solution; physically, it means that the
    solution ``u`` *radiates energy towards infinity*.

Let us begin by specifying the partial differential equation, and creating the
domain, mesh, and quadrature for the problem:

```@example getting_started
using Inti, LinearAlgebra, StaticArrays

# PDE
k = 2π
pde = Inti.Helmholtz(; dim = 2, k)

# Create the geometry as the union of a kite and a circle
kite = Inti.parametric_curve(0.0, 1.0) do s
    return SVector(2.5 + cos(2π * s[1]) + 0.65 * cos(4π * s[1]) - 0.65, 1.5 * sin(2π * s[1]))
end
circle = Inti.parametric_curve(0.0, 1.0) do s
    return SVector(cos(2π * s[1]), sin(2π * s[1]))
end
Γ = kite ∪ circle
# Create a mesh for the geometry
msh = Inti.meshgen(Γ; meshsize = 2π / k / 10)
Q = Inti.Quadrature(msh; qorder = 5)
```

We can easily check the mesh by visualizing it using the `Meshes.jl` package:

```@example getting_started
using Meshes, GLMakie
fig, ax, pl = viz(msh; segmentsize = 3, axis = (aspect = DataAspect(), ))
```

Next we need to reformulate the Helmholtz problem as a boundary integral
equation. Among the plethora of options, we will use in this tutorial a simple
*direct* formulation, which uses Green's third identity to relate the values of
``u`` and ``\partial_{\nu} u`` on ``\Gamma``:

```math
    -\frac{u(\boldsymbol{x})}{2} + D[u](\boldsymbol{x}) = S[\partial_\nu u](\boldsymbol{x}), \quad \boldsymbol{x} \in \Gamma.
```

Here ``S`` and ``D`` are the single- and double-layer operators, formally
defined as:

```math
    S[\sigma](\boldsymbol{x}) = \int_\Gamma G(\boldsymbol{x}, \boldsymbol{y}) \sigma(\boldsymbol{y}) \ \mathrm{d}\Gamma(\boldsymbol{y}), \quad
    D[\sigma](\boldsymbol{x}) = \int_\Gamma \frac{\partial G}{\partial \nu_{\boldsymbol{y}}}(\boldsymbol{x}, \boldsymbol{y}) \sigma(\boldsymbol{y}) \ \mathrm{d}\Gamma(\boldsymbol{y}),
```

where ``G`` is the fundamental solution of the Helmholtz equation. Note that
``G`` is typically singular when ``\boldsymbol{x} = \boldsymbol{y}``, and
therefore the numerical discretization of these integral operators requires
special care.

To approximate ``S`` and ``D`` in Inti.jl we can proceed as follows:

```@example getting_started
S, D = Inti.single_double_layer(;
    pde,
    target = Q,
    source = Q,
    compression = (method = :none,),
    correction = (method = :dim,),
)
```

Much of the complexity involved in the numerical computation is hidden in the
function above; later in the tutorial we will discuss in more details the
options available for the *compression* and *correction* methods, as well as how
to define your own kernels and operators. For now, it suffices to know that `S`
and `D` are matrix-like objects that can be used to solve the boundary integral
equation. For that, we need to provide the boundary data ``g``. 

We are interested in the scattered field ``u`` produced by an incident plane
wave ``u_i = e^{i k \boldsymbol{d} \cdot \boldsymbol{x}}``, where
``\boldsymbol{d}`` is a unit vector denoting the direction of the plane wave.
Assuming that the total field ``u_t = u_i + u`` satisfies a homogenous Neumann
condition on ``\Gamma``, and that the scattered field ``u`` satisfies the
Sommerfeld radiation condition, we can write the boundary condition as:

```math
    \partial_\nu u = -\partial_\nu u_i, \quad \boldsymbol{x} \in \Gamma.
```

We can thus solve the boundary integral equation to find ``u`` on ``\Gamma``:

```@example getting_started
# define the incident field and compute its normal derivative
θ = 0
d = SVector(cos(θ), sin(θ))
g = map(Q) do q
    # normal derivative of e^{ik*d⃗⋅x}
    x, ν = q.coords, q.normal
    return -im * k * exp(im * k * dot(x, d)) * dot(d, ν)
end ## Neumann trace on boundary
u = (-I / 2 + D) \ (S * g) # Dirichlet trace on boundary
```

Now that we know both the Dirichlet and Neumann data on the boundary, we can use
Green's representation formula, i.e., 

```math
    \mathcal{D}[u](\boldsymbol{r}) - \mathcal{S}[\partial_{\nu} u](\boldsymbol{r}) = \begin{cases}
        u(\boldsymbol{r}) & \text{if } \boldsymbol{r} \in \mathbb{R}^2 \setminus \overline{\Omega},\\
        0 & \text{if } \boldsymbol{r} \in \Omega,
    \end{cases}
```

to compute the solution ``u`` in the domain:

```@example getting_started
𝒮, 𝒟 = Inti.single_double_layer_potential(; pde, source = Q)
uₛ = x -> 𝒟[u](x) - 𝒮[g](x)
```

To wrap things up, let's visualize the scattered field:

```@example getting_started
using GLMakie # or your favorite plotting backend for Makie
xx = yy = range(-5; stop = 5, length = 100)
U = map(uₛ, Iterators.product(xx, yy))
Ui = map(x -> exp(im*k*dot(x, d)), Iterators.product(xx, yy))
Ut = Ui + U
fig, ax, hm = heatmap(
    xx,
    yy,
    real(Ut);
    colormap = :inferno,
    interpolate = true,
    axis = (aspect = DataAspect(), xgridvisible = false, ygridvisible = false),
)
viz!(msh; segmentsize = 2)
Colorbar(fig[1, 2], hm; label = "real(u)")
fig # hide
```

!!! tip "Going further"
    - ...

```@example getting_started
# build an exact solution
G = Inti.SingleLayerKernel(pde)
dG = Inti.DoubleLayerKernel(pde)
xs = map(θ -> 0.5 * rand() * SVector(cos(θ), sin(θ)), 2π * rand(10))
cs = rand(ComplexF64, length(xs))
uₑ  = q -> sum(c * G(x, q) for (x, c) in zip(xs, cs))
∂ₙu = q -> sum(c * dG(x, q) for (x, c) in zip(xs, cs))
g  = map(∂ₙu, Q) 
u = (-I / 2 + D) \ (S * g)
uₛ = x -> 𝒟[u](x) - 𝒮[g](x)
pts = [5*SVector(cos(θ), sin(θ)) for θ in range(0, 2π, length = 100)]
er = norm(uₛ.(pts) - uₑ.(pts), Inf)
println("maximum error on circle of radius 5: $er")
```
