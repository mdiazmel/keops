# The convenient `kernel_product` helper - pytorch only

On top of the low-level syntax, we also provide
a **kernel name parser** that lets you quickly define and work with
most of the kernel products used in shape analysis.
This high-level interface relies on two operators:

```python
from pykeops.torch import Kernel, kernel_product
```

- `Kernel` is the name parser: it turns a string identifier (say, `"gaussian(x,y) * (1 + linear(u,v)**2 )"`) into a set of KeOps formulas.
- `kernel_product` is the "numerical" torch routine. It takes as input a dict of parameters and a set of input tensors, to return a fully differentiable torch variable.

## Example : Gaussian convolution on a vector space

A quick example: here is how you
can compute a *fully differentiable* Gaussian-RBF kernel product:

```python
import torch
from pykeops.torch import Kernel, kernel_product

# Generate the data as pytorch tensors
x = torch.randn(1000,3, requires_grad=True)
y = torch.randn(2000,3, requires_grad=True)
b = torch.randn(2000,2, requires_grad=True)

# Pre-defined kernel: using custom expressions is also possible!
# Notice that the parameter sigma is a dim-1 vector, *not* a scalar:
sigma  = torch.tensor([.5], requires_grad=True)
params = {
    "id"      : Kernel("gaussian(x,y)"),
    "gamma"   : 1./sigma**2,
}

# Depending on the inputs' types, 'a' is a CPU or a GPU variable.
# It can be differentiated wrt. x, y, b and sigma.
a = kernel_product(params, x, y, b)
```

## (Gaussian * Cauchy-Binet) varifold kernel on a product space

Before going into details, let's showcase a slightly longer computation: that of
a **Cauchy-Binet varifold kernel** on the space of points+orientations.
Given:

- a set $`(x_i)`$ of target points in $`\mathbb{R}^3`$;
- a set $`(u_i)`$ of target orientations in $`\mathbb{S}^1`$, encoded as unit-norm vectors in $`\mathbb{R}^3`$;
- a set $`(y_j)`$ of source points in $`\mathbb{R}^3`$;
- a set $`(v_j)`$ of source orientations in $`\mathbb{S}^1`$, encoded as unit-norm vectors in $`\mathbb{R}^3`$;
- a set $`(b_j)`$ of source signal values in $`\mathbb{R}^4`$;

we will compute the "target" signal values

```math
 a_i ~=~  \sum_j K(\,x_i,u_i\,;\,y_j,v_j\,)\,\cdot\, b_j ~=~ \sum_j k(x_i,y_j)\cdot \langle u_i, v_j\rangle^2 \cdot b_j,
```

where $`k(x_i,y_j) = \exp(-\|x_i - y_j\|^2 / \sigma^2)`$.

```python
import torch
import torch.nn.functional as F
from pykeops.torch import Kernel, kernel_product

N, M = 1000, 2000 # number of "i" and "j" indices
# Generate the data as pytorch tensors.

# First, the "i" variables:
x = torch.randn(N,3) # Positions,    in R^3
u = torch.randn(N,2) # Orientations, in R^2 (for example)

# Then, the "j" ones:
y = torch.randn(M,3) # Positions,    in R^3
v = torch.randn(M,2) # Orientations, in R^2

# The signal b_j, supported by the (y_j,v_j)'s
b = torch.randn(M,4)

# Pre-defined kernel: using custom expressions is also possible!
# Notice that the parameter sigma is a dim-1 vector, *not* a scalar:
sigma  = torch.tensor([.5])
params = {
    # The "id" is defined using a set of special function names
    "id"      : Kernel("gaussian(x,y) * (linear(u,v)**2) "),
    # gaussian(x,y) requires a standard deviation; linear(u,v) requires no parameter
    "gamma"   : ( 1./sigma**2 , None ) ,
}

# Don't forget to normalize the orientations:
u = F.normalize(u, p=2, dim=1)
v = F.normalize(v, p=2, dim=1)

# We're good to go! Notice how we grouped together the "i" and "j" features:
a = kernel_product(params, (x,u), (y,v), b)
```

## The `Kernel` parser

The cornerstone of our high-level syntax is the `Kernel` constructor, that takes as input a **string** name and returns a pre-processed kernel identifier. A valid kernel name is built from pre-defined formulas, acting on arbitrary pairs of variables, and combined using:

- integer constants, 
- the addition `+`, 
- the product `*`,
- the integer exponentiation `**k`.

A kernel name is thus associated to a list of *formulas* (that will require **parameters**) and to a list of **pairs of variables**, ordered as they are in the name string. Both of them will be required as inputs by `kernel_product`. A few examples:

- `"gaussian(x,y)"` : one formula and one pair of variables.
- `"gaussian(x,y) * linear(u,v)**2"` : two formulas and two pairs of variables.
- `"cauchy(x,y) + gaussian(x,y) * (1 + cauchy(u,v)**2)` : **three** formulas (`cauchy`, `gaussian` and `cauchy` once again) with **two** pairs of variables (`(x,y)` first, `(u,v)` second)

Note that by convention, pairs of variables should be denoted by single-letter, non-overlapping duets - `"gaussian(x',yy)"` or `"gaussian(x,y) + cauchy(y,z)"` are not supported.

### Atomic formulas available

As of today, the pre-defined kernel names are:

- `linear(x,y)` $`= \langle x,y\rangle`$, the L² scalar product;
- `gaussian(x,y)` $`= \exp(-\langle x-y, G\, (x-y)\rangle)`$, the standard RBF kernel;
- `laplacian(x,y)` $`= \exp(-\sqrt{\langle x-y, G\, (x-y)\rangle})`$, the exponential pointy kernel;
- `cauchy(x,y)` $`= 1/(1+\langle x-y, G\, (x-y)\rangle)`$, a heavy-tail kernel;
- `inverse_multiquadric(x,y)` $`= 1/\sqrt{1+\langle x-y, G\, (x-y)\rangle}`$, a very heavy-tail kernel;
- `-distance(x,y)` $`= - \sqrt{\langle x-y, G\, (x-y)\rangle}`$, generates the Energy Distance between measures;

**Defining your own formulas** is possible, and documented in the second part of the [kernel_product_syntax](https://plmlab.math.cnrs.fr/benjamin.charlier/libkeops/blob/master/pykeops/examples/kernel_product_syntax.py) example.

### Atomic formulas' parameters

With the exception of the linear kernel (which accepts `None` as its parameter), all these kernels act on arbitrary vectors of dimension D and are parametrized by a variable `G` that can represent :

- a scalar, if `G` is a dim-1 vector;
- a diagonal matrix, if `G` is a dim-D vector;
- a symmetric D-by-D matrix, if `G` is a dim-D*D vector.
- a j-varying scalar, if `G` is an M-by-1 tensor;
- a j-varying diagonal matrix, if `G` is an M-by-D tensor;
- a j-varying symmetric D-by-D matrix, if `G` is an M-by-D*D tensor.

If required by the user, a kernel-id can thus be used to represent non-uniform, non-radial kernels as documented in the [anisotropic_kernels example](https://plmlab.math.cnrs.fr/benjamin.charlier/libkeops/blob/master/pykeops/examples/anisotropic_kernels.py).

## The `kernel_product` routine

Having created our kernel-id, and with a few torch tensors at hand, we can feed the `kernel_product` numerical routine with the appropriate input. More precisely, if `Kernel("my_kernel_name...")` defines a kernel with **F formulas** and **V variable pairs**, `kernel_product` will accept the following arguments:

- a `parameters` dict with the following entries:
  + `"id" = Kernel("my_kernel_name...")` - **mandatory**: the kernel id, as documented above.
  + `"gamma" = (G_0, G_1, ..., G_(F-1))` - **mandatory**: a list or tuple of formula parameters - one per formula. As documented above, each of them can be either `None`, a torch vector or a torch 2D tensor. Note that if F=1, we also accept the use of `"gamma" = G_0` instead of `(G_0,)`.
  + `"backend" = ["auto"] | "pytorch" | "CPU" | "GPU" | "GPU_1D" | "GPU_2D"` - optional: the same set of options as `generic_sum` with an additionnal **pure-vanilla-pytorch** backend that does rely on the KeOps engine.
  + `"mode" = ["sum"] | "lse"` - optional : the **operation** performed on the data. The possible choices are documented below.
- a tuple `(X_0, ..., X_(V-1))` of torch tensors, with the same size N along the dimension 0. Note that if V=1, we also accept `X_0` in place of `(X_0,)`.
- a tuple `(Y_0, ..., Y_(V-1))` of torch tensors, with the same size M along the dimension 0. We should have `X_k.size(1) == Y_k.size(1)` for 0 <= k <= V-1. Note that if V=1, we also accept `Y_0` in place of `(Y_0,)`.
- a torch tensor `B` of shape M-by-E.
- (optional:) a keyword argument `mode = ["sum"] | "lse"`, whose value supersedes that of `parameters["mode"]`.

Then, provided that these conditions are satisfied,

```python
a = kernel_product( { "id"    : Kernel("my_kernel..."),
                      "gamma" : (G_0, G_1, ..., G_(F-1)),
                      "backend" : "auto",
                      "mode"    : "sum",    },
                      (X_0,...,X_(V-1)), (Y_0,...,Y_(V-1)), B,   mode = "sum" )
```

defines a fully-differentiable N-by-E torch tensor:

```math
 a_i ~=~  \sum_j \text{my\_kernel}_{G_0, G_1, ...}(\,x^0_i,x^1_i,...\,;\,y^0_j,y^1_j,...\,) \,\cdot\, b_j,
```

where the kernel parameters $`G_k`$ may possibly be indexed by "j".

### Kernel modes

Kernel computations are not limited to simple kernel products. Therefore, we provide a high-level interface for the following operations:

- If `mode == "sum"`,

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), B, mode = "sum" )
  ```

  allows us to compute:

  ```math
  a_i ~=~  \sum_j K_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,) \,\cdot\, b_j.
  ```

- If `mode == "lse"`,

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), B, mode = "lse" )
  ```

  allows us to compute (with numerically stable computations):

  ```math
  a_i ~=~  \log \sum_j \exp \big( \log(K)_{G_0, ...}(\,x^0_i,...\,;\,y^0_j,...\,) \,+\, b_j \big).
  ```

- If `mode == "log_scaled"`, `kernel_products` accepts two additional tensor parameters `U` (N-by-1) and `V` (M-by-1), so that

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), B, U, V, mode = "log_scaled" )
  ```
  
  allows us to compute:

  ```math
  a_i ~=~  \sum_j \exp \big( \log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,)\,+\,u_i\,+\,v_j\big)\,\cdot\, b_j.
  ```

- If `mode == "log_scaled_lse"`, `kernel_products` accepts two additional tensor parameters `U` (N-by-1) and `V` (M-by-1), so that

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), B, U, V, mode = "log_scaled_lse" )
  ```

  allows us to compute:

  ```math
  a_i ~=~  \log \sum_j \exp \big( \log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,)\,+\,u_i\,+\,v_j\,+\, b_j\big).
  ```

- If `mode == "log_scaled_barycenter"`, `kernel_products` accepts three additional tensor parameters `U` (N-by-1), `V` (M-by-1) and `C` (N-by-E), so that

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), B, U, V, C, mode = "log_scaled_barycenter" )
  ```

  allows us to compute:

  ```math
  a_i ~=~  \sum_j \exp \big( \log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,)\,+\,u_i\,+\,v_j\big)\,\cdot\, (b_j-c_i).
  ```

- If `mode == "lse_mult_i"`, `kernel_products` accepts an additional tensor parameter `H` (N-by-1), so that

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), B, H, mode = "lse_mult_i" )
  ```

  allows us to compute:

  ```math
  a_i ~=~  \log \sum_j \exp \big( \,h_i\cdot\log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,)\,+\,b_j\big).
  ```

- If `mode == "sinkhorn_cost"`, `kernel_products` accepts two tensor parameters `S` (N-by-1) and `T` (M-by-1) **instead** of `B`, so that

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), S, T, mode = "sinkhorn_cost" )
  ```

  allows us to compute:

  ```math
  a_i ~=~  \sum_j -\log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,) \,\cdot\, \exp \big( \log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,)\,+\,s_i\,+\,t_j\big).
  ```

- If `mode == "sinkhorn_primal"`, `kernel_products` accepts four tensor parameters `S` (N-by-1), `T` (M-by-1), `U` (N-by-1) and `V` (M-by-1) **instead** of `B`, so that

  ```python
  a = kernel_product( params, (X_0,...), (Y_0,...), S, T, U, V, mode = "sinkhorn_primal" )
  ```

  allows us to compute:

  ```math
  a_i ~=~  \sum_j (u_i+v_j-1)\,\cdot\, \exp \big( \log(K)_{G_0,...}(\,x^0_i,...\,;\,y^0_j,...\,)\,+\,s_i\,+\,t_j\big).
  ```

**If you think that other kernel-operations should be supported, feel free to ask!**
