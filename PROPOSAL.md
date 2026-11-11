# Squishy at Scale: Parallel XPBD Soft-Body Simulation — Proposal

**Isabella Kwon**

**URL:** https://github.com/yujinkwon03/SoftEgg *(URL subject to change)*

## Summary

I will implement the substepped XPBD solver of Müller et al. [1] and apply it to 2D soft bodies (specified by triangle meshes) that collide with each other and with rigid bodies. The simulation will be written in C++ and parallelized with (1) OpenMP on multicore processors and (2) CUDA on NVIDIA GPUs. I will then compare the performance of each stage within a substep across the sequential and parallel versions.

## Background

Müller et al. [1] describe a position-based solver for rigid bodies built on XPBD (Extended Position-Based Dynamics). It predicts where bodies will move and then corrects those predictions so that constraints are satisfied. The authors describe this method as unconditionally stable: it does not face the same risk of explosion as naive explicit force-based integrators.

Each constraint C(x) is corrected with:

```
Δλ = (−C − α̃λ) / (Σ wᵢ|∇ᵢC|² + α̃)
xᵢ += wᵢ ∇ᵢC Δλ
λ  += Δλ
```

wᵢ is particle i's inverse mass, ∇ᵢC is the constraint gradient for particle i, α̃ = α/h² is the scaled compliance (inverse stiffness), and λ is the accumulated correction.

My implementation applies the paper's solver to soft bodies made of particles, so there is no particle rotation. Each soft body is a triangle mesh constrained by (1) edge lengths, (2) triangle areas, and (3) contacts on its boundary. Each substep consists of stages that must run in order:

```
Algorithm 1: Substepped XPBD loop (per frame)

broad phase      candidate body pairs (AABBs enlarged by 2·Δt·v_max)
h = Δt / numSubsteps
for each substep:
  predict        prev = x;  v += h·g;  x += h·v
  find contacts  boundary vertices vs. candidate bodies' edges and rigid
                 shapes; store pre-solve normal velocity
  solve          λ = 0; one iteration over edge, triangle-area, contact
                 (incl. static friction) and floor constraints
  update         v = (x − prev) / h
  velocity pass  dynamic friction, restitution, damping
```

I will focus on this substep for this final project. There is ample opportunity for parallelism in this solver, as every substep processes tens of thousands of particles, constraints, and contacts, many of which are independent. But it is not as simple as parallelizing over particles: there are dependencies that make this problem more interesting to parallelize (see The Challenge).

## The Challenge

**Shared particles**
- Neighboring constraints move the same particle (in a triangulated mesh, a vertex can be shared by 6 triangles), so solving constraints in parallel causes write conflicts.
- The paper's solver applies each correction immediately (Gauss-Seidel), which is inherently sequential. A parallel version must accumulate corrections and apply them afterward (Jacobi).

**Substep stage dependencies**
- The stages of each substep must run in order, because each stage reads the previous stage's writes to neighboring particles.

**Irregular collisions**
- Contacts change every substep. For example, if bodies pile up, a small region holds most of the contacts, which can cause load imbalance and contention on the same particles.

**Memory-bound workload**
- The solver is likely limited by memory bandwidth and contention rather than computation: particles are accessed through index lists, with relatively little arithmetic per access. A key goal is to diagnose these bottlenecks and find the best approach to mitigating them.

## Platform Choice

To avoid overhead from Unity's background tasks and to time accurately, I will implement the solver in C++, which also gives me more control over memory layout. OpenMP lets me parallelize across multiple cores and compare scheduling and atomics. CUDA on an NVIDIA GPU (GHC machines / personal computer) appears to fit the problem well: there are tens of thousands of simple computations per substep, so GPU parallelization is a natural fit.

Visual debugging will happen in Unity and C#, but this is not central to the solver and is not included in any timing.

## Resources

**Machines**
- GHC machines / personal laptop (NVIDIA GPU) for CUDA, benchmarks and Unity visualization.

**Code**
- Written from scratch in C++/CUDA. My earlier Unity C# prototype (one explicit-spring soft ball) is used only as a visual reference.
- Although implemented in a different language, this resource is a useful point of reference: [Physics-Based Simulation — Position Based Dynamics](https://phys-sim-book.github.io/position-based-simulations/position-based-dynamics-pbd/position_based_dynamics.html)

**References**

[1] M. Müller, M. Macklin, N. Chentanez, S. Jeschke, T.-Y. Kim. *Detailed Rigid Body Simulation with Extended Position Based Dynamics.* Computer Graphics Forum (SCA), 2020.

## Goals and Deliverables

### Plan to achieve

1. Sequential C++ solver
   - Edge and triangle-area constraints
   - Floor/walls, soft–soft and soft–rigid contacts
   - Friction/restitution velocity pass
2. OpenMP solver
   - Comparing atomics vs. per-particle gathering for shared particles
3. CUDA GPU solver
   - Comparing atomics vs. per-particle gathering for shared particles
4. Benchmarks for all three solvers on sample scenes with varying compositions (to be specified once the sequential solver is complete): many vs. few collisions, large meshes, etc.
5. Bottleneck analysis using high-watermark experiments (removing atomics, removing math) to show whether each stage is limited by memory, synchronization or computation.

I can't yet give a specific target speedup. While building the sequential solver, I will measure an upper bound on what it can compute in real time: for each sample scene, the largest size that runs at 60 FPS with 10 substeps. I will then record how much larger these scenes can grow with the parallel solvers while staying in real time.

### Stretch goals

- Graph-colored Gauss-Seidel compared against Jacobi
- Interactive live demo: a complex scene with many soft bodies influenced by user input

### If behind schedule

- Use ring-based bodies instead of triangle meshes, or include only the floor and walls as rigid obstacles.

## Schedule

| Week | Milestone |
|---|---|
| 11/9 – 11/15 | Create sequential solver in C++. Mesh generation, edge and triangle-area constraint data handled in C#. Lay foundations for data passing between Unity and C++. Begin contacts (soft–soft and soft–rigid). |
| 11/16 – 11/18 | Finish sequential solver implementation, including collision-pair broad phase. Benchmark program for the sequential solver: record per-stage, whole-substep and whole-simulation (including initialization) time. |
| 11/18 – 11/22 | Begin OpenMP (likely continues into next week). |
| 11/23 – 11/29 | Finish OpenMP implementation (atomics vs. gather). Begin CUDA implementation. |
| 11/30 – 12/6 | Milestone report. Finish CUDA implementation. |
| 12/7 – 12/10 | Create graphs from experiments. Finalize report. |
