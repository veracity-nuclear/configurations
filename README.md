# configurations

Contains configuration files for containers, spack, build scripts, etc.

This repository provides [Spack](https://spack.io/) environment files for building the dependencies required by nuclear simulation codes (SCALE, MPACT, VERA) on HPC clusters.

---

## Table of Contents

- [Repository Layout](#repository-layout)
- [Installing Spack](#installing-spack)
- [Setting Up a Spack Environment](#setting-up-a-spack-environment)
- [Using the Environment](#using-the-environment)
- [Available Environments](#available-environments)

---

## Repository Layout

```
spack/
├── mpact/          # MPACT dependency environments
├── scale/          # SCALE dependency environments
└── vera/           # VERA dependency environments
```

Each subdirectory contains one or more `spack.yaml` files. File names encode the target machine, compiler, and MPI library, e.g. `spack_teton_gcc13_openmpi.yaml`.

---

## Installing Spack

> **Prerequisites:** `git`, `curl` (or `wget`), a C/C++/Fortran compiler, and Python 3.6+.

### 1. Clone Spack

```bash
git clone -c feature.manyFiles=true https://github.com/spack/spack.git ~/spack
```

### 2. Source the setup script

Add the following to your `~/.bashrc` (or `~/.bash_profile`) so Spack is available in every shell session:

```bash
# Spack
export SPACK_ROOT=~/spack
source $SPACK_ROOT/share/spack/setup-env.sh
```

Then reload your shell:

```bash
source ~/.bashrc
```

### 3. Verify the installation

```bash
spack --version
```

### 4. (Optional) Find compilers and system packages

```bash
spack compiler find          # auto-detect compilers on PATH
spack external find          # detect commonly installed system packages
```

> **On HPC clusters:** load the required compiler module *before* running `spack compiler find` so that Spack registers it. For example:
> ```bash
> module load gcc/13.4.0
> spack compiler find
> ```

---

## Setting Up a Spack Environment

Spack environments are fully described by the YAML files in this repository. The general workflow is:

### 1. Choose a YAML file

Pick the file that matches your machine and desired compiler/MPI combination, e.g.:

| File | Machine | Compiler | MPI |
|------|---------|----------|-----|
| `spack/scale/spack_teton_gcc13_openmpi.yaml` | Teton | GCC 13.4.0 | OpenMPI 5.0.9 |
| `spack/scale/spack_teton_gcc13_mpich.yaml` | Teton | GCC 13.4.0 | MPICH 4.3.2 |
| `spack/scale/spack_sawtooth_gcc13_openmpi.yaml` | Sawtooth | GCC 13.4.0 | OpenMPI 4.1.6 |
| `spack/scale/spack_windriver_gcc13_openmpi.yaml` | Windriver | GCC 13.3.0 | OpenMPI 4.1.6 |
| `spack/vera/vera_sawtooth_4-4_openmpi_gcc8.4.0.yaml` | Sawtooth | GCC 8.4.0 | OpenMPI 3.1.6 |
| `spack/vera/vera_sawtooth_4-4_mpich_gcc8.4.0.yaml` | Sawtooth | GCC 8.4.0 | MPICH 4.2.3 |
| `spack/mpact/aaron_mpact_4-4.yaml` | — | GCC 8.4.0 | MPICH 3.4 |
| `spack/mpact/aaron_mpact_gpu.yaml` | — | GCC 11.4.0 | OpenMPI 4.1.5 |

### 2. (HPC only) Load prerequisite modules

Several YAML files assume that a compiler and/or MPI module is already loaded. Check the comments at the top of the file you are using. For example:

```bash
# spack_teton_gcc13_mpich.yaml requires:
module load gcc/13.4.0-gcc11-5dyu
module load mpich/4.3.2-gcc13-iljv
```

### 3. Create a named environment from the YAML file

```bash
spack env create my-env path/to/spack_<machine>_<compiler>_<mpi>.yaml
```

For example:

```bash
spack env create scale-teton spack/scale/spack_teton_gcc13_openmpi.yaml
```

### 4. Activate the environment

```bash
spack env activate scale-teton
```

Your shell prompt will change to indicate the active environment.

### 5. Concretize the environment

Concretization resolves the full dependency graph and checks for conflicts before anything is built:

```bash
spack concretize
```

Review the output to confirm that all specs look correct.

### 6. Install all packages

```bash
spack install
```

This step can take a significant amount of time on the first run. Subsequent installs reuse cached builds. Use the `-j` flag to control parallelism if Spack did not pick up the `build_jobs` setting from the YAML:

```bash
spack install -j 12
```

---

## Using the Environment

### Activate

```bash
spack env activate <env-name>
```

Once activated, all installed executables and libraries in the environment's view are available on `PATH`/`LD_LIBRARY_PATH` automatically (because the YAML files set `view: true`).

### Deactivate

```bash
spack env deactivate
```

### Load an individual package (without activating the whole environment)

```bash
spack load hdf5
```

### Check what is installed

```bash
spack find
```

### Use environment variables in build systems

With the environment active, CMake-based projects can find Spack-installed packages through standard CMake find-package machinery. A typical workflow:

```bash
spack env activate scale-teton

mkdir build && cd build
cmake .. \
  -DCMAKE_PREFIX_PATH=$(spack location -i hdf5) \
  -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

Alternatively, export Spack's view root directly:

```bash
export CMAKE_PREFIX_PATH=$(spack location --env scale-teton -i)
```

### Reproduce an environment on another machine

The YAML file is fully self-contained. Copy it to the new machine and repeat steps 3–6 above. Pin exact package hashes by running:

```bash
spack env activate <env-name>
spack concretize
spack env depfile > Makefile   # optional: parallel build via make
```

---

## Adding or Modifying Packages

1. Edit the `specs:` list in the YAML file.
2. Re-concretize:
   ```bash
   spack env activate <env-name>
   spack concretize -f   # -f forces re-concretization
   spack install
   ```

## Useful Spack Commands Reference

| Command | Description |
|---------|-------------|
| `spack env list` | List all named environments |
| `spack env status` | Show the currently active environment |
| `spack find` | List installed packages in the active environment |
| `spack spec <pkg>` | Show the full spec (dependencies) for a package |
| `spack uninstall <pkg>` | Remove a package |
| `spack gc` | Remove unused packages (garbage collect) |
| `spack clean -a` | Clear all build caches |

For full documentation see <https://spack.readthedocs.io>.
