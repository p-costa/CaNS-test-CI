<!--- the logo -->
<img src="assets/img/CaNS-logo.png" height=100>

## Synopsis

**CaNS (Canonical Navier-Stokes)** is a code for massively-parallel numerical simulations of fluid flows. It aims at solving any fluid flow of an incompressible, Newtonian fluid that can benefit from a FFT-based solver for the second-order finite-difference Poisson equation in a 3D Cartesian grid. In two directions the grid is regular and the solver supports the following combination of (homogeneous) boundary conditions:

 * Neumann-Neumann
 * Dirichlet-Dirichlet
 * Neumann-Dirichlet
 * Periodic

In the third domain direction, the solver is more flexible as it uses Gauss elimination. There the grid can also be non-uniform (e.g. fine at the boundary and coarser in the center).

CaNS also allows for choosing an implicit temporal discretization of the momentum diffusion terms, either fully implicit or only along the last domain direction. This results in solving a 3D/1D Helmholtz equation per velocity component. In the fully implicit case, FFT-based solvers are also used, and the same options described above for pressure boundary conditions apply to the velocity.

**Reference**

P. Costa. *A FFT-based finite-difference solver for massively-parallel direct numerical simulations of turbulent flows.* *Computers & Mathematics with Applications* 76: 1853--1862 (2018). [doi:10.1016/j.camwa.2018.07.034](https://doi.org/10.1016/j.camwa.2018.07.034) [[arXiv preprint]](https://arxiv.org/abs/1802.10323)

## News

### _Major Update:_ `CaNS 3.0` _is out!_ :tada:
See the [Release Notes](https://github.com/CaNS-World/CaNS/releases/tag/v3.0.0) for more details.

**[10/04/2025]:** The writing of checkpoint files has changed. To allow for more flexibility, CaNS now writes one file per scalar field, where each velocity component, pressure, and scalar fields is stored in a different checkpoint file.

**[14/02/2025]:** Most pre-processor macros have been turned into runtime arguments, which allow for a much simpler control of the computational setup without re-compiling the source. See the updated [`docs/INFO_INPUT.md`](docs/INFO_INPUT.md) and [`docs/INFO_COMPILING.md`](docs/INFO_COMPILING.md) for more details.

**[08/02/2025]:** A new approach for solving the Poisson equation based on a Parallel Cyclic Reduction -- Tridiagonal Matrix Algorithm (PCR-TDMA) approach. See [`docs/INFO_COMPILING.md`](docs/INFO_COMPILING.md) for more details.

**[31/01/2025]:** CaNS now can solve the transport equations associated with an arbritrary number of scalars :tada:, with a versatile discretization of the diffusion term. Thermal convection under the Boussinesq approximation is also supported. See the updated [`docs/INFO_INPUT.md`](docs/INFO_INPUT.md) and [`docs/INFO_COMPILING.md`](docs/INFO_COMPILING.md) files for more details, and some of the new cases under `examples/`.

**[29/01/2025]:** To avoid having the `build.conf` and `configs/*.mk` files - often changed by the user - tracked by git, the compiling behavior has changed slightly; see Compilation below.

**[28/01/2025]:** Input file `input.nml` has changed slightly, to allow for prescribing a fixed time step size `dt_f`.

## Features

Some features are:

 * Hybrid MPI/OpenMP parallelization
 * FFTW guru interface / cuFFT used for computing multi-dimensional vectors of 1D transforms
 * The right type of transformation (Fourier, cosine, sine, etc) is automatically determined form the input file
 * [cuDecomp](https://github.com/NVIDIA/cuDecomp) pencil decomposition library for _hardware-adaptive_ distributed memory calculations on _many GPUs_
 * [2DECOMP&FFT](https://github.com/xcompact3d/2decomp-fft) library used for performing global data transpositions on CPUs and some of the data I/O
 * GPU acceleration using OpenACC directives
 * A different canonical flow can be simulated just by changing the input files

Some examples of flows that this code can solve are:

 * periodic or developing channel
 * periodic or developing square duct
 * tri-periodic domain
 * lid-driven cavity

## Motivation

This project aimed first at being a modern alternative to the well-known FISHPACK routines (Paul Swarztrauber & Roland Sweet, NCAR) for solving a three-dimensional Helmholtz equation. After noticing some works simulating canonical flows with iterative solvers -- when faster direct solvers could have been used instead -- it seemed natural to create a versatile tool and make it available. This code can be used as a first base code for which solvers for more complex flows can be developed (e.g. extensions with fictitious domain methods).

## Method

The fluid flow is solved with a second-order finite difference pressure correction scheme, discretized in a MAC grid arrangement. Time is advanced with a three-step low storage Runge-Kutta scheme. Optionally, for increased stability at low Reynolds numbers, at the price of higher computational demand, the diffusion term can be treated implicitly. See the reference above for details.

## Usage

### Downloading *CaNS*

Since *CaNS* loads the external pencil decomposition libraries as Git Submodules, the repository should be cloned as follows:
```bash
git clone --recursive https://github.com/CaNS-World/CaNS
```
so the libraries are downloaded too. Alternatively, in case the repository has already been cloned without the Submodules (i.e., folders `cuDecomp` and `2decomp-fft` under `dependencies/` are empty), the following command can be used to update them:
```bash
git submodule update --init --recursive
```

### Compilation

#### Prerequisites
The prerequisites for compiling CaNS are the following:

 * MPI
 * FFTW3/cuFFT library for CPU/GPU runs
 * The `nvfortran` compiler (for GPU runs)
 * CMake for compiling the cuDecomp library (for GPU runs)
 * NCCL and NVSHMEM (optional, may be exploited by the cuDecomp library)
 * OpenMP (optional)

#### In short
For most systems, CaNS can be compiled from the root directory with the following commands `make libs && make`, which will compile the 2DECOMP&FFT/cuDecomp libraries, and CaNS.

#### Detailed instructions
The `Makefile` in root directory is used to compile the code, and is expected to work out-of-the-box for most systems. The `build.conf` file in the root directory can be used to choose the Fortran compiler (MPI wrapper), and a few pre-defined profiles depending on the nature of the run (e.g., production vs debugging), and pre-processing options; see [`INFO_COMPILING.md`](docs/INFO_COMPILING.md) for more details. The default `build.conf` file is created from `configs/defaults/build-default.conf` at the first compilation. Concerning the pre-processing options, the following are available:

 * `SINGLE_PRECISION` : calculation will be carried out in single precision (the default precision is double)
 * `GPU`              : enable GPU-accelerated runs

### Input file

The input file `input.nml` sets the physical and computational parameters. In the `examples/` folder are examples of input files for several canonical flows. See [`INFO_INPUT.md`](docs/INFO_INPUT.md) for a detailed description of the input file.

Files `out1d.h90`, `out2d.h90` and `out3d.h90` in `src/` set which data are written in 1-, 2- and 3-dimensional output files, respectively. *The code should be recompiled after editing out?d.h90 files*.

### Running the code

Run the executable with `mpirun` with a number of tasks complying to what has been set in the input file `dns.in`. Data will be written by default in a folder named `data/`, which must be located where the executable is run (by default in the `run/` folder).

### Visualizing field data

See [`INFO_VISU.md`](docs/INFO_VISU.md).

## Contributing

We appreciate any contributions and feedback that can improve CaNS. If you wish to contribute to the tool, please get in touch with the maintainers or open an Issue in the repository / a thread in Discussions. Pull Requests are welcome, but please propose/discuss the changes in an linked Issue first.

## Final notes

Please read the `ACKNOWLEDGEMENTS`, `LICENSE` files.
