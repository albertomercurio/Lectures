# QuantumToolbox.jl
Alberto Mercurio

## Introduction

[QuantumToolbox.jl](https://github.com/qutip/QuantumToolbox.jl) was born
during my Ph.D., driven by the need for a high-performance framework for
quantum simulations. At the time, I was already using
[QuTiP](https://github.com/qutip/qutip) (Quantum Toolbox in Python)

<iframe src="https://qutip.org" width="100%" height="500px"></iframe>

However, I was lloking for a more efficient solution. I initially
explored
[QuantumOptics.jl](https://github.com/qojulia/QuantumOptics.jl), but its
syntax differed significantly from QuTiP’s, which made the transition
challenging. Motivated by the desire for both performance and
familiarity, as well as a deep curiosity to learn Julia, I decided to
develop my own package.

## A Demo Code: the Schrödinger Equation

Let’s consider a quantum harmonic oscillator with Hamiltonian
($\hbar = 1$)

$$
\hat{H} = \omega_0 \hat{a}^\dagger \hat{a} \, ,
$$

and start from the state

$$
\left| \psi(0) \right\rangle = \frac{1}{\sqrt{2}} \left( \left| 2 \right\rangle + \left| 3 \right\rangle \right) \, .
$$

We now want to solve the Schrödinger equation

$$
i \frac{d}{dt} \left| \psi(t) \right\rangle = \hat{H} \left| \psi(t) \right\rangle \, .
$$

This can easily be done with QuTiP using the `sesolve` function. We also
want to compute the expectation value of the position operator

$$
\left\langle \hat{a} + \hat{a}^\dagger \right\rangle (t) = \left\langle \psi(t) \right| \hat{a} + \hat{a}^\dagger \left| \psi(t) \right\rangle \, .
$$

An analytical solution is known,

$$
\vert \psi (t) \rangle = \frac{1}{\sqrt{2}} \left( e^{-i 2 \omega_0 t} \vert 2 \rangle + e^{-i 3 \omega_0 t} \vert 3 \rangle \right) \, ,
$$

and so

$$
\langle \hat{a} + \hat{a}^\dagger \rangle (t) = \sqrt{3} \cos (\omega_0 t) \, ,
$$

and we can compare the numerical results with it.

### The QuTiP case

``` python
import numpy as np
from qutip import *

N = 10 # cutoff for Fock states
a = destroy(N)
H = a.dag() * a

psi0 = (fock(N, 2) + fock(N, 3)).unit()
tlist = np.linspace(0, 10, 100)
e_ops = [a + a.dag()]

result = sesolve(H, psi0, tlist, e_ops=e_ops)
```

### QuantumToolbox.jl: Almost the same syntax

``` julia
using QuantumToolbox

N = 10
a = destroy(N)
H = a' * a

psi0 = (fock(N, 2) + fock(N, 3)) |> normalize
tlist = range(0, 10, 100)
e_ops = [a + a']

result = sesolve(H, psi0, tlist, e_ops=e_ops)
```

    Progress: [==============================] 100.0% --- Elapsed Time: 0h 00m 00s (ETA: 0h 00m 00s)

    Solution of time evolution
    (return code: Success)
    --------------------------
    num_states = 1
    num_expect = 1
    ODE alg.: OrdinaryDiffEqTsit5.Tsit5{typeof(OrdinaryDiffEqCore.trivial_limiter!), typeof(OrdinaryDiffEqCore.trivial_limiter!), Static.False}(OrdinaryDiffEqCore.trivial_limiter!, OrdinaryDiffEqCore.trivial_limiter!, static(false))
    abstol = 1.0e-8
    reltol = 1.0e-6

And we can plot the results using
[Makie.jl](https://github.com/MakieOrg/Makie.jl) for example

``` julia
using CairoMakie

fig = Figure(size=(700, 300), fontsize=15)
ax = Axis(fig[1, 1], xlabel="Time", ylabel=L"\langle \hat{a} + \hat{a}^\dagger \rangle")

lines!(ax, result.times, real.(result.expect[1,:]), linewidth=3, label="Numerical")
lines!(ax, result.times, sqrt(3) .* cos.(result.times), linewidth=3, label="Analytical", linestyle=:dash)

xlims!(ax, result.times[1], result.times[end])

axislegend(ax)

fig
```

<img src="introduction_files/figure-commonmark/cell-3-output-1.png"
width="700" height="300" />

## The `QuantumObject` struct

If we take a look at the structure of the annihilation operator
$\hat{a}$, we can see that it is a
[`QuantumObject`](https://qutip.org/QuantumToolbox.jl/stable/resources/api#QuantumToolbox.QuantumObject)
Julia constructor.

``` julia
typeof(a)
```

    QuantumObject{SparseMatrixCSC{ComplexF64, Int64}, OperatorQuantumObject, 1}

``` julia
a
```

    Quantum Object:   type=Operator   dims=[10]   size=(10, 10)   ishermitian=false
    10×10 SparseMatrixCSC{ComplexF64, Int64} with 9 stored entries:
         ⋅      1.0+0.0im          ⋅      …          ⋅          ⋅    
         ⋅          ⋅      1.41421+0.0im             ⋅          ⋅    
         ⋅          ⋅              ⋅                 ⋅          ⋅    
         ⋅          ⋅              ⋅                 ⋅          ⋅    
         ⋅          ⋅              ⋅                 ⋅          ⋅    
         ⋅          ⋅              ⋅      …          ⋅          ⋅    
         ⋅          ⋅              ⋅                 ⋅          ⋅    
         ⋅          ⋅              ⋅         2.82843+0.0im      ⋅    
         ⋅          ⋅              ⋅                 ⋅      3.0+0.0im
         ⋅          ⋅              ⋅                 ⋅          ⋅    

A `QuantumObject` struct is defined as follows

``` julia
struct QuantumObject{MT<:AbstractArray,ObjType<:QuantumObjectType,N} <: AbstractQuantumObject{MT,ObjType,N}
    data::MT
    type::ObjType
    dims::SVector{N,Int}
end
```

The `data` field contains the actual data of the quantum object, in this
case it is a sparse matrix. This follows from the definition of the
matrix elements of the annihilation operator

$$
\langle n \vert \hat{a} \vert m \rangle = \sqrt{m} \ \delta_{n, m-1} \, ,
$$

where we defined $N$ as the cutoff for the Fock states. The `type` field
gives the type of the quantum object

- `Ket` for ket states
- `Bra` for bra states
- `Operator` for operators
- `SuperOperator` for superoperators (e.g., Liouvillian)
- `OperatorKet` for vectorized representation of operators, acting as a
  ket state
- `OperatorBra` for vectorized representation of operators, acting as a
  bra state

Finally, the `dims` field contains the list of dimensions of the Hilbert
spaces. Its length is equal to the number of subsystems, and each
element is the dimension of the corresponding subsystem.

## Large Hilbert space dimensions: the need for GPU acceleration

The example above was quite simple, where an analytical solution was
known. However, in many cases, the system is more complex and even the
numerical solution can be challenging. For instance, the Hilbert space
dimension can be very large when considering many subsystems. Let’s make
a practical example by considering a 2-dimensional transferse field
Ising model with 3x2 spins. The Hamiltonian is given by

The total Hamiltonian is

$$
\hat{H} = \frac{J_z}{2} \sum_{\langle i,j \rangle} \hat{\sigma}_i^z \hat{\sigma}_j^z + h_x \sum_i \hat{\sigma}_i^x \, ,
$$

and, since we are including losses, the time evolution of the density
matrix is governed by the Lindblad master equation

$$
\frac{d}{d t} \hat{\rho} = \mathcal{L}[\hat{\rho}] = -i[\hat{H}, \hat{\rho}] + \sum_k \left( \hat{L}_k \hat{\rho} \hat{L}_k^\dagger - \frac{1}{2} \{\hat{L}_k^\dagger \hat{L}_k, \hat{\rho}\} \right) \, ,
$$

with the dissipators

$$
\hat{L}_k = \sqrt{\gamma} \hat{\sigma}_k^- \, ,
$$

and $\gamma$ the decay rate.

### The vectorized representation of the density matrix

The Liouvillian $\mathcal{L}$ is a superoperator, meaning that it acts
on operators. A convenient way to represent its action is by vectorizing
the density matrix

$$
\hat{\rho} =
\begin{pmatrix}
\rho_{11} & \rho_{12} & \cdots & \rho_{1N} \\
\rho_{21} & \rho_{22} & \cdots & \rho_{2N} \\
\vdots & \vdots & \ddots & \vdots \\
\rho_{N1} & \rho_{N2} & \cdots & \rho_{NN}
\end{pmatrix}
\rightarrow
\begin{pmatrix}
\rho_{11} \\
\rho_{21} \\
\vdots \\
\rho_{N1} \\
\rho_{12} \\
\rho_{22} \\
\vdots \\
\rho_{N2} \\
\vdots \\
\rho_{1N} \\
\rho_{2N} \\
\vdots \\
\rho_{NN}
\end{pmatrix} \, .
$$