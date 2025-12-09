# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of Python-based solid simulation tutorials focused on optimization-based methods with algorithmic convergence guarantees. The codebase demonstrates penetration-free and inversion-free simulations using implicit Euler time integration and the projected Newton method. Each numbered directory represents a self-contained tutorial example that builds on concepts from previous examples.

Complementary theory can be found in the free online book [Physics-based Simulation](https://phys-sim-book.github.io/).

## Running Examples

Each example directory (0-11) is self-contained with its own `simulator.py` and dependencies.

### Standard Simulation Examples (0-9)

**Dependencies:**
```bash
pip install numpy scipy pygame
```

**Run any example:**
```bash
cd <example_directory>
python simulator.py
```

Examples include:
- `0_getting_started/` - Symplectic Euler integration basics (simulator0.py, simulator1.py, simulator2.py)
- `1_mass_spring/` - Mass-spring system with implicit Euler
- `2_dirichlet/` - Dirichlet boundary conditions
- `3_contact/` - Ground contact with barrier energy
- `4_friction/` - Friction on slopes
- `5_mov_dirichlet/` - Moving Dirichlet boundaries
- `6_inv_free/` - Inversion-free neo-Hookean elasticity
- `7_self_contact/` - Self-contact with IPC
- `8_self_friction/` - Self-friction
- `9_reduced_DOF/` - Reduced-order simulation with modal analysis

### MPM Examples (10-11)

**Dependencies:**
```bash
pip install numpy taichi
```

**Run:**
```bash
cd <10_mpm_elasticity or 11_mpm_sand>
python simulator.py
```

- `10_mpm_elasticity/` - Material Point Method for elastic collisions
- `11_mpm_sand/` - MPM for granular materials

## Code Architecture

### Energy-Based Simulation Framework

All examples follow an energy minimization approach where each simulation step solves:
```
minimize: Incremental Potential (IP) = Inertia + h² × (Elasticity + External Forces)
```

**Core Components Per Example:**

1. **Energy Modules** (e.g., `InertiaEnergy.py`, `MassSpringEnergy.py`, `NeoHookeanEnergy.py`)
   - Each implements `val()`, `grad()`, `hess()` functions
   - Compute energy value, gradient, and Hessian in IJV (COO) format
   - `hess()` returns `[I_indices, J_indices, values]` for sparse matrix assembly

2. **Time Integrator** (`time_integrator.py`)
   - `step_forward()`: Main time-stepping function using implicit Euler
   - `IP_val()`, `IP_grad()`, `IP_hess()`: Incremental Potential calculations
   - `search_dir()`: Solves linear system for Newton search direction
   - Newton loop with backtracking line search until convergence

3. **Simulator** (`simulator.py`)
   - Sets up simulation parameters (density, stiffness, time step size)
   - Initializes mesh using `square_mesh.py`
   - Main loop: visualization with pygame and time integration

4. **Mesh Generation** (`square_mesh.py`)
   - `generate()`: Creates node positions and connectivity
   - `write_to_file()`: Outputs mesh for analysis

5. **Utilities** (`utils.py`)
   - `make_PSD()`: Projects Hessian to positive semi-definite for stability

### Progressive Feature Additions

Each example builds on the previous with new energy terms:

- **1-2**: Basic elasticity (MassSpring or NeoHookean) + Inertia
- **3-4**: + GravityEnergy + BarrierEnergy (contact)
- **4**: + FrictionEnergy
- **5-6**: Inversion-free elasticity with moving boundaries
- **7-8**: Self-contact using `distance/` modules (PointPointDistance, PointEdgeDistance, CCD)
- **9**: Reduced-order with `linear.py` for modal analysis

### Key Algorithms

**Projected Newton Method:**
```python
# Newton loop in time_integrator.py
while residual > tolerance:
    p = search_dir(...)  # Solve: H·p = -grad
    alpha = line_search(...)  # Backtracking with energy decrease check
    x += alpha * p
```

**Barrier Energy (Contact):**
- Located in `BarrierEnergy.py` (examples 3-9)
- `init_step_size()`: Conservative step size to prevent interpenetration
- Log-barrier function ensures collision-free simulation

**Dirichlet Boundary Conditions:**
- Applied in `search_dir()` by zeroing rows/columns of projected Hessian
- `is_DBC` array marks constrained nodes

**Reduced DOF (example 9):**
- `linear.py`: Modal analysis to compute eigenmodes
- `reduced_basis`: Projection matrix for reduced space
- Chain rule applied in `search_dir()` to transform gradient/Hessian

## Development Patterns

### Adding New Energy Terms

1. Create `<Name>Energy.py` with `val()`, `grad()`, `hess()` functions
2. Import in `time_integrator.py`
3. Add to `IP_val()`, `IP_grad()`, `IP_hess()` with appropriate `h²` scaling
4. For contact-like terms, update `init_step_size()` if needed

### Modifying Time Integration

- Time step size `h` is critical for stability (typically 0.001-0.01)
- Tolerance `tol` controls Newton convergence (typically 1e-2 to 1e-4)
- Line search ensures energy decrease each iteration

### Working with Mesh Data

- Node positions: `x` is list of 2D numpy arrays `[[x, y], ...]`
- Edges/elements: `e` is list of node index pairs/tuples
- Velocity: `v` has same structure as `x`
- Mass: `m` is list of scalars per node

## Code Anchors

Some files contain `# ANCHOR: <name>` and `# ANCHOR_END: <name>` comments marking important code sections referenced in documentation. Preserve these when editing.

Examples:
- `3_contact/time_integrator.py:26-31`: filter_ls (filtered line search)
- `9_reduced_DOF/time_integrator.py:59-73`: search_dir (reduced space projection)
