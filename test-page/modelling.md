---
// title: Numerical Modelling of Seismic Waves
// author: Mathias Louboutin, and Felix J. Herrmann
bibliography:
	- modelling.bib	
	- slimbib.bib
---

### Figure: {#forward-adjoint-wave.fig .wide}
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/Forward.gif){#fwd width=45%}
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/Adjoint.gif){#bck width=45%}
A 2D view of a seismic wave propagating forward #fwd and backward #bck through the standard marmousi model.

Since its re-introduction by [@Pratt], full-waveform inversion (FWI) has gained a lot of attention in geophysical exploration because of its ability to build high resolution velocity models more or less automatically in areas of complex geology. However, the core components of FWI is the ability to model the solutiuon of the wave-equation fast for non trivial representations of the physics. 

At SLIM, we have always focused on the development of high-level abstractiosn to facilitate research and development in exploration geophysics. This effort has led to the inseption of [Devito], a domain specific language (DSL) for finite-differences and more generally stencil computation [@devito-compiler;@devito-api]. Additionally, we have built a seismic inversion framework in Julia that uses [Devito] as its computational backend. This framework, called the Julia Devito Inversion Framework [JUDI][@witteJUDI2019]\, is a Linear Algebra based DSL that allow easy and high-level implementation of seismic inversion basedx on the linear operator representation of the wave-equation and related operators.

In the following, we introduce seismic modelling through the [Devito] DSL. This introduction sets ground for wave-equation based geophysical exploration using [JUDI].

## Introduction

Devito provides a concise and straightforward computational framework for discretizing wave equations, which underlie all FWI frameworks. We will show that it generates verifiable executable code at run time for wave propagators associated with forward and (in part 2) adjoint wave equations. Devito frees the user from the recurrent and time-consuming development of performant time-stepping codes and allows the user to concentrate on the geophysics of the problem rather than on low-level implementation details of wave-equation simulators. This tutorial covers the conventional adjoint-state formulation of full-waveform tomography [@Tarantola] that underlies most of the current methods referred to as full-waveform inversion [@Virieux]\. While other formulations have been developed to improve the convergence of FWI for poor starting models, in these tutorials we will concentrate on the standard formulation that relies on the combination of a forward/adjoint pair of propagators and a correlation-based gradient. In part one of this tutorial, we discuss how to set up wave simulations for inversion, including how to express the wave equation in Devito symbolically and how to deal with the acquisition geometry.

<div style="padding: 12px 12px 18px 12px; margin: 20px 0px 20px 0px; background: #eeeeff; border: 2px solid #8888aa; border-radius: 4px;">
<h4>What is FWI?</h4>
<p>FWI tries to iteratively minimize the difference between data that was acquired in a seismic survey and synthetic data that is generated from a wave simulator with an estimated (velocity) model of the subsurface. As such, each FWI framework essentially consists of a wave simulator for forward modeling the predicted data and an adjoint simulator for calculating a model update from the data misfit. This first part of this tutorial is dedicated to the forward modeling part and demonstrates how to discretize and implement the acoustic wave equation using Devito.</p>
</div>


## Wave simulations for inversion

The acoustic wave equation with the squared slowness ``m``, defined as ``m(x,y)=c^{-2}(x,y)`` with ``c(x,y)`` being the unknown spatially varying wavespeed, is given by:

```math {#wave}
m(x, y) \frac{\mathrm{d}^2 u(t, x, y)}{\mathrm{d}t^2}\ -\ \Delta u(t, x, y)\ +\ \eta(x, y) \frac{\mathrm{d} u(t, x, y)}{\mathrm{d}t}\ \  =\ \  q(t, x, y; x_\mathrm{s}, y_\mathrm{s}),\ \ \ \ \ \ \ \ (1)
```

where ``\Delta`` is the Laplace operator, ``q(t, x, y;x_\mathrm{s}, y_\mathrm{s})`` is the seismic source, located at ``(x_\mathrm{s}, y_\mathrm{s})`` and ``\eta(x, y)`` is a space-dependent dampening parameter for the absorbing boundary layer [@Cerjan]\. As shown in Figure 1, the physical model is extended in every direction by `nbpml` grid points to mimic an infinite domain. The dampening term ``\eta\, \mathrm{d}u/\mathrm{d}t`` attenuates the waves in the dampening layer and prevents waves from reflecting at the model boundaries. In Devito, the discrete representations of ``m`` and ``\eta`` are contained in a `model` object that contains a `grid` object with all relevant information such as the origin of the coordinate system, grid spacing, size of the model and dimensions `time, x, y`:

#### Figure: {#setup}
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/Figure1_composed.png)
: (a) Diagram showing the model domain, with the perfectly matched layer (PML) as an absorbing layer to attenuate the wavefield at the model boundary. (b) The example model used in this tutorial, with the source and receivers indicated. The grid lines show the cell boundaries.


```python
# Define a velocity model. The velocity is in km/s
vp = np.empty((101, 101), dtype=np.float32)
vp[:, :51] = 1.5
vp[:, 51:] = 2.5
model = Model(vp=vp,             # A velocity model.
              origin=(0, 0),     # Top left corner.
              shape=(101, 101),  # Number of grid points.
              spacing=(10, 10),  # Grid spacing in m.
              nbpml=40)          # boundary layer.
```


```python
# Quick plot of model.
plot_velocity(model)
```


#### Figure: {#fig:model}
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/TLE_Forward_6_0.png)
: Two layer velocity model    


In the `Model` instantiation, `vp` is the velocity in ``\text{km}/\text{s}``, `origin` is the origin of the physical model in meters, `spacing` is the discrete grid spacing in meters, `shape` is the number of grid points in each dimension and `nbpml` is the number of grid points in the absorbing boundary layer. Is is important to note that `shape` is the size of the physical domain only, while the total number of grid points, including the absorbing boundary layer, will be automatically derived from `shape` and `nbpml`.

## Symbolic definition of the wave propagator

To model seismic data by solving the acoustic wave equation, the first necessary step is to discretize this partial differential equation (PDE), which includes discrete representations of the velocity model and wavefields, as well as approximations of the spatial and temporal derivatives using finite-differences (FD). Unfortunately, implementing these finite-difference schemes in low-level code by hand is error prone, especially when we want performant and reliable code. 

The primary design objective of Devito is to allow users to define complex matrix-free finite-difference approximations from high-level symbolic definitions, while employing automated code generation to create highly optimized low-level C code. Using the symbolic algebra package SymPy [@Meurer17] to facilitate the automatic creation of derivative expressions, Devito generates computationally efficient wave propagators.

At the core of Devito's symbolic API are symbolic types that behave like SymPy function objects, while also managing  data:

* `Function` objects represent a spatially varying function discretized on a regular Cartesian grid. For example, a function symbol `f = Function(name='f', grid=model.grid, space_order=2)` is denoted symbolically as `f(x, y)`. The objects provide auto-generated symbolic expressions for finite-difference derivatives through shorthand expressions like `f.dx` and `f.dx2` for the first and second derivative in `x`.

* `TimeFunction` objects represent a time-dependent function that has ``\text{time}`` as the leading dimension, for example `g(time, x, y)`. In addition to spatial derivatives `TimeFunction` symbols also provide time derivatives `g.dt` and `g.dt2`.

* `SparseFunction` objects represent sparse components, such as sources and receivers, which are usually distributed sparsely and often located off the computational grid — these objects also therefore handle interpolation onto the model grid.

To demonstrate Devito's symbolic capabilities, let us consider a time-dependent function ``\mathbf{u}(\text{time}, x, y)`` representing the discrete forward wavefield:


```python
t0 = 0.     # Simulation starts a t=0
tn = 1000.  # Simulation last 1 second (1000 ms)
dt = model.critical_dt  # Time step from model grid spacing

nt = int(1 + (tn-t0) / dt)  # Discrete time axis length
time = np.linspace(t0, tn, nt)  # Discrete modelling time
u = TimeFunction(name="u", grid=model.grid, 
                 time_order=2, space_order=2,
                 save=True, time_dim=nt)
```

where the `grid` object provided by the `model` defines the size of the allocated memory region, `time_order` and `space_order` define the default discretization order of the derived derivative expressions.

We can now use this symbolic representation of our wavefield to generate simple discretized expressions for finite-difference derivative approximations using shorthand expressions, such as `u.dt` and `u.dt2` to denote ``\frac{\text{d} u}{\text{d} t}`` and ``\frac{\text{d}^2 u}{\text{d} t^2}`` respectively. Using the automatic derivation of derivative expressions, we can now implement a discretized expression for Equation #wave without the source term ``q(x,y,t;x_s, y_s)``. The `model` object, which we created earlier, already contains the squared slowness ``\mathbf{m}`` and damping term ``\mathbf{\eta}`` as `Function` objects:


```python
pde = model.m * u.dt2 - u.laplace + model.damp * u.dt
```

If we write out the (second order) second time derivative `u.dt2` as shown earlier and ignore the damping term for the moment, our `pde` expression translates to the following discrete the wave equation:

```math {#pde1}
 \frac{\mathbf{m}}{\text{dt}^2} \Big( \mathbf{u}[\text{time}-\text{dt}] - 2\mathbf{u}[\text{time}] + \mathbf{u}[\text{time}+\text{dt}]\Big) - \Delta \mathbf{u}[\text{time}] = 0, \quad \text{time}=1 \cdots n_{t-1} \ \ \ \ \ \ \ (2)
```

with ``\text{time}`` being the current time step and ``\text{dt}`` being the time stepping interval. To propagate the wavefield, we rearrange to obtain an expression for the wavefield ``\mathbf{u}(\text{time}+\text{dt})`` at the next time step. Ignoring the damping term once again, this yields:

```math {#pde2}
 \mathbf{u}[\text{time}+\text{dt}] = 2\mathbf{u}[\text{time}] - \mathbf{u}[\text{time}-\text{dt}] + \frac{\text{dt}^2}{\mathbf{m}} \Delta \mathbf{u}[\text{time}] \ \ \ \ \ \ \ (3)
```

We can rearrange our `pde` expression automatically using the SymPy utility function `solve`, then create an expression which defines the update of the wavefield for the new time step ``\mathbf{u}(\text{time}+\text{dt})``, with the command `u.forward`:

```python
stencil = Eq(u.forward, solve(pde, u.forward))
```

`stencil` represents the finite-difference approximation derived from Equation #pde2, including the finite-difference approximation of the Laplacian and the damping term. Although it defines the update for a single time step only, Devito knows that we will be solving a time-dependent problem over a number of time steps because the wavefield `u` is a `TimeFunction` object.


## Setting up the acquisition geometry

The expression for time stepping we derived in the previous section does not contain a seismic source function yet, so the update for the wavefield at a new time step is solely defined by the two previous wavefields. However as indicated in Equation #wave\, wavefields for seismic experiments are often excited by an active (impulsive) source ``q(x,y,t;x_\text{s})``, which is a function of space and time (just like the wavefield `u`). To include such a source term in our modeling scheme, we simply add the the source wavefield as an additional term to Equation #pde2\:

```math {#eqsrc}
 \mathbf{u}[\text{time}+\text{dt}] = 2\mathbf{u}[\text{time}] - \mathbf{u}[\text{time}-\text{dt}] + \frac{\text{dt}^2}{\mathbf{m}} \Big(\Delta \mathbf{u}[\text{time}] + \mathbf{q}[\text{time}]\Big). \ \ \ \ \ \ \ (4)
```

Since the source appears on the right-hand side in the original equation (Equation #wave), the term also needs to be multiplied with ``\frac{\text{dt}^2}{\mathbf{m}}`` (this follows from rearranging Equation #pde1, with the source on the right-hand side in place of 0). Unlike the discrete wavefield `u` however, the source `q` is typically localized in space and only a function of time, which means the time-dependent source wavelet is injected into the propagating wavefield at a specified source location. The same applies when we sample the wavefield at  receiver locations to simulate a shot record, i.e. the simulated wavefield needs to be sampled at specified receiver locations only. Source and receiver both do not necessarily coincide with the modeling grid.

Here, `RickerSource` acts as a wrapper around `SparseFunction` and models a Ricker wavelet with a peak frequency `f0` and source coordinates `src_coords`:


```python
f0 = 0.010  # kHz, peak frequency.
src = RickerSource(name='src', grid=model.grid, f0=f0,
                   time=time, coordinates=src_coords)
```

The `src.inject` function now injects the current time sample of the Ricker wavelet (weighted with ``\frac{\text{dt}^2}{\mathbf{m}}`` as shown in Equation #eqsrc) into the updated wavefield `u.forward` at the specified coordinates.


```python
src_term = src.inject(field=u.forward,
                      expr=src * dt**2 / model.m,
                      offset=model.nbpml)
```

To extract the wavefield at a predetermined set of receiver locations, there is a corresponding wrapper function for receivers as well, which creates a `SparseFunction` object for a given number `npoint` of receivers, number `nt` of time samples, and specified receiver coordinates `rec_coords`:

```python
rec = Receiver(name='rec', npoint=101, ntime=nt,
               grid=model.grid, coordinates=rec_coords)
```

Rather than injecting a function into the model as we did for the source, we now simply save the wavefield at the grid points that correspond to receiver positions and interpolate the data to their exact possibly of the computatational grid location:


```python
rec_term = rec.interpolate(u, offset=model.nbpml)
```


### Figure: {#geom}    
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/TLE_Forward_29_0.png)
: Acquisiton geometry

With the acquistion geometry defined we can now define a propagator.

## Forward simulation 

We can now define our forward propagator by adding the source and receiver terms to our stencil object:

```python
op_fwd = Operator([stencil] + src_term + rec_term)
```

The symbolic expressions used to create `Operator` contain sufficient meta-information for Devito to create a fully functional computational kernel. The dimension symbols contained in the symbolic function object (`time, x, y`) define the loop structure of the created code,while allowing Devito to automatically optimize the underlying loop structure to increase execution speed.

The size of the loops and spacing between grid points is inferred from the symbolic `Function` objects and associated `model.grid` object at run-time. As a result, we can invoke the generated kernel through a simple Python function call by supplying the number of timesteps `time` and the timestep size `dt`. The user data associated with each `Function` is updated in-place during operator execution, allowing us to extract the final wavefield and shot record directly from the symbolic function objects without unwanted memory duplication:


```python
op_fwd(time=nt, dt=model.critical_dt)
```

When this has finished running, the resulting wavefield is stored in `u.data` and the shot record is in `rec.data`. We can easily plot this 2D array as an image, as shown in Figure 2.

#### Figure: {#shot}    
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/TLE_Forward_36_0.png)
: The shot record generated by Devito for the example velocity model.

As demonstrated in the notebook, a movie of snapshots of the forward wavefield can also be generated by capturing the wavefield at discrete time steps. Figure 3 shows three timesteps from the movie.


#### Figure: {#snap}  
![](https://slim.gatech.edu/Website-ResearchWebInfo/Modelling/Figures/TLE_Forward_39_0.png)
: Three time steps from the wavefield simulation that resulted in the shot record in Figure 2. You can generate an animated version in the Notebook at github.com/seg.


<video width="576" height="432" controls autoplay loop>
  <source type="video/mp4" src="data:video/mp4;base64,AAAAHGZ0eXBNNFYgAAACAGlzb21pc28yYXZjMQAAAAhmcmVlAAFt321kYXQAAAKvBgX//6vcRem9
5tlIt5Ys2CDZI+7veDI2NCAtIGNvcmUgMTQ4IHIyNjY4IGZkMmMzMjQgLSBILjI2NC9NUEVHLTQg
QVZDIGNvZGVjIC0gQ29weWxlZnQgMjAwMy0yMDE2IC0gaHR0cDovL3d3dy52aWRlb2xhbi5vcmcv
eDI2NC5odG1sIC0gb3B0aW9uczogY2FiYWM9MSByZWY9MyBkZWJsb2NrPTE6MDowIGFuYWx5c2U9
MHgzOjB4MTEzIG1lPWhleCBzdWJtZT03IHBzeT0xIHBzeV9yZD0xLjAwOjAuMDAgbWl4ZWRfcmVm
PTEgbWVfcmFuZ2U9MTYgY2hyb21hX21lPTEgdHJlbGxpcz0xIDh4OGRjdD0xIGNxbT0wIGRlYWR6
b25lPTIxLDExIGZhc3RfcHNraXA9MSBjaHJvbWFfcXBfb2Zmc2V0PS0yIHRocmVhZHM9MTIgbG9v
a2FoZWFkX3RocmVhZHM9MiBzbGljZWRfdGhyZWFkcz0wIG5yPTAgZGVjaW1hdGU9MSBpbnRlcmxh
Y2VkPTAgYmx1cmF5X2NvbXBhdD0wIGNvbnN0cmFpbmVkX2ludHJhPTAgYmZyYW1lcz0zIGJfcHly
YW1pZD0yIGJfYWRhcHQ9MSBiX2JpYXM9MCBkaXJlY3Q9MSB3ZWlnaHRiPTEgb3Blbl9nb3A9MCB3
ZWlnaHRwPTIga2V5aW50PTI1MCBrZXlpbnRfbWluPTIwIHNjZW5lY3V0PTQwIGludHJhX3JlZnJl
c2g9MCByY19sb29rYWhlYWQ9NDAgcmM9Y3JmIG1idHJlZT0xIGNyZj0yMy4wIHFjb21wPTAuNjAg
cXBtaW49MCBxcG1heD02OSBxcHN0ZXA9NCBpcF9yYXRpbz0xLjQwIGFxPTE6MS4wMACAAAAUgmWI
hAA///73aJ8Cm1pDeoDklcUl20+B/6tncHyP6QMAAAMAAAMAAIDgO/WSlFvtbKQAAA8pyn9x7mfg
ATnU+hFeHwwkughZQkKzKou2H8K0jnVwShlBIJOkzpS5HextmEG4MBIDutQ38tyai3d6MvtoO8Tn
6gNFsqH8QgOiSP5aIaDc7oAeGbYVCv8kGqhKa83PpEF1Q1AiKNh0VktYY/nbR7oNB1EQi0qKdsp1
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
AAABAAAXPgAABAAAAQAAAAAHNG1kaWEAAAAgbWRoZAAAAAAAAAAAAAAAAAAAKAAAAO4AVcQAAAAA
AC1oZGxyAAAAAAAAAAB2aWRlAAAAAAAAAAAAAAAAVmlkZW9IYW5kbGVyAAAABt9taW5mAAAAFHZt
aGQAAAABAAAAAAAAAAAAAAAkZGluZgAAABxkcmVmAAAAAAAAAAEAAAAMdXJsIAAAAAEAAAafc3Ri
bAAAALNzdHNkAAAAAAAAAAEAAACjYXZjMQAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAJAAbAASAAA
AEgAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABj//wAAADFhdmNDAWQA
Fv/hABhnZAAWrNlAkDehAAADAAEAAAMAKA8WLZYBAAZo6+PLIsAAAAAcdXVpZGtoQPJfJE/Fujml
G88DI/MAAAAAAAAAGHN0dHMAAAAAAAAAAQAAAHcAAAIAAAAAFHN0c3MAAAAAAAAAAQAAAAEAAAOY
Y3R0cwAAAAAAAABxAAAAAQAABAAAAAABAAAKAAAAAAEAAAQAAAAAAQAAAAAAAAABAAACAAAAAAEA
AAoAAAAAAQAABAAAAAABAAAAAAAAAAEAAAIAAAAAAQAACgAAAAABAAAEAAAAAAEAAAAAAAAAAQAA
AgAAAAABAAAKAAAAAAEAAAQAAAAAAQAAAAAAAAABAAACAAAAAAEAAAoAAAAAAQAABAAAAAABAAAA
AAAAAAEAAAIAAAAAAQAABAAAAAABAAAKAAAAAAEAAAQAAAAAAQAAAAAAAAABAAACAAAAAAEAAAoA
AAAAAQAABAAAAAABAAAAAAAAAAEAAAIAAAAAAQAACgAAAAABAAAEAAAAAAEAAAAAAAAAAQAAAgAA
AAABAAAKAAAAAAEAAAQAAAAAAQAAAAAAAAABAAACAAAAAAEAAAoAAAAAAQAABAAAAAABAAAAAAAA
AAEAAAIAAAAAAQAACgAAAAABAAAEAAAAAAEAAAAAAAAAAQAAAgAAAAABAAAKAAAAAAEAAAQAAAAA
AQAAAAAAAAABAAACAAAAAAEAAAoAAAAAAQAABAAAAAABAAAAAAAAAAEAAAIAAAAAAQAACgAAAAAB
AAAEAAAAAAEAAAAAAAAAAQAAAgAAAAADAAAEAAAAAAEAAAoAAAAAAQAABAAAAAABAAAAAAAAAAEA
AAIAAAAAAQAACAAAAAACAAACAAAAAAMAAAQAAAAAAQAACAAAAAACAAACAAAAAAEAAAQAAAAAAQAA
BgAAAAABAAACAAAAAAEAAAQAAAAAAQAABgAAAAABAAACAAAAAAEAAAYAAAAAAQAAAgAAAAABAAAG
AAAAAAEAAAIAAAAAAQAABgAAAAABAAACAAAAAAEAAAYAAAAAAQAAAgAAAAABAAAGAAAAAAEAAAIA
AAAAAQAABgAAAAABAAACAAAAAAEAAAYAAAAAAQAAAgAAAAABAAAGAAAAAAEAAAIAAAAAAQAABgAA
AAABAAACAAAAAAEAAAYAAAAAAQAAAgAAAAABAAAGAAAAAAEAAAIAAAAAAQAABgAAAAABAAACAAAA
AAEAAAYAAAAAAQAAAgAAAAABAAAGAAAAAAEAAAIAAAAAAQAABgAAAAABAAACAAAAAAEAAAYAAAAA
AQAAAgAAAAABAAAGAAAAAAEAAAIAAAAAAQAABgAAAAABAAACAAAAAAEAAAYAAAAAAQAAAgAAAAAB
AAAEAAAAABxzdHNjAAAAAAAAAAEAAAABAAAAdwAAAAEAAAHwc3RzegAAAAAAAAAAAAAAdwAAFzkA
AACoAAAANAAAACYAAAAmAAAAsgAAAEwAAAAvAAAAQgAAARgAAABnAAAARgAAAGsAAAF9AAAAzQAA
AHUAAACCAAABwgAAATQAAAC0AAAA1wAAAhcAAAJXAAABxwAAASsAAAEkAAADNQAAAd8AAAE8AAAB
UgAAA7MAAAIPAAABcwAAAbgAAAQlAAACgwAAAccAAAHlAAAEJAAAAsMAAAIHAAACNwAABRkAAAM+
AAACdgAAAn0AAAUgAAADRAAAAnwAAAKvAAAF+gAAA9sAAAMCAAACswAABwoAAAQmAAAC9QAAA0kA
AAUrAAAFaQAABc0AAAiCAAAERgAAAyMAAAOcAAAHLgAABC0AAAM8AAAFigAABXQAAAWeAAAHQgAA
BBMAAALyAAAFSAAABcIAAAM+AAAE0AAABYgAAALSAAAFiQAAAtQAAAWrAAACvAAABTUAAAKEAAAF
VgAAAkMAAAUSAAAB7wAABO0AAAHTAAAE5gAAAasAAATDAAAB4QAABNoAAAG8AAAEhwAAAYMAAAPI
AAABpQAAA9AAAAFWAAADbwAAATkAAAOWAAAA1QAAA70AAADlAAADeQAAAMoAAAM+AAAA6gAAAyYA
AADGAAADIQAAAOsAAAKNAAAAFHN0Y28AAAAAAAAAAQAAACwAAABidWR0YQAAAFptZXRhAAAAAAAA
ACFoZGxyAAAAAAAAAABtZGlyYXBwbAAAAAAAAAAAAAAAAC1pbHN0AAAAJal0b28AAAAdZGF0YQAA
AAEAAAAATGF2ZjU3LjI1LjEwMA==
">
  Your browser does not support the video tag.
</video>

## Ongoing research

With [Devito] we can now easily define and model complex physics representations such as the TTI or elastic wave-equations. One of the main complexity arising from these advanced representation is the extended computational and memory cost of inversion. The core research direction in seismic modeling relies on machine learning to design discretization and that allow to work on much coarser grid while consering high accuracy. This work is inspired by recent advanced in the field of computational Fluid Dynamics [@Brenner] where the simpler case of diffusive or Elliptique equation can be solved on coarser grid using learned disretizations. 


## References


<hr />
## Supplemental material

- [Devito documentation](http://www.opesci.org/)
- [Devito source code and examples](https://github.com/opesci/Devito)
- [Tutorial notebooks](https://github.com/slimgroup/Devito-Examples)
- [JUDI homepage](https://github.com/slimgroup/JUDI.jl)

<hr> 


[Devito]:https://github.com/opesci/Devito
[JUDI]:https://github.com/slimgroup/JUDI.jl