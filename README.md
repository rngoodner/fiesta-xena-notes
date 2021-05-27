# FIESTA Xena notes

Notes on how to build/run [FIESTA](https://github.com/CUP-ECS/fiesta) on [Xena](https://carc.unm.edu/systems/Systems1.html) at [UNM CARC](https://carc.unm.edu).

# Helpful resources

- [CARC HPC documentation](https://github.com/UNM-CARC/QuickBytes/blob/master/README.md)
- [Lmod documentation](https://lmod.readthedocs.io/en/latest/010_user.html)
- [Spack documentation](https://spack.readthedocs.io/en/latest/)
- [Singularity documentation](https://sylabs.io/guides/3.7/user-guide/)

# Build using Lmod for dependencies

Note that all the versions of hdf5 available on Xena via Lmod require intel-mpi.

- `git clone https://github.com/CUP-ECS/fiesta.git`
- `cd fiesta`
- `mkdir build`
- `cd build`
- `module load gcc intel-mpi cuda cmake hdf5`
- `cmake .. -DCUDA=on`
- `make -j`

# Build using Spack for dependencies

Using Spack will enable us to build hdf5 for openmpi.
Note that Xena has a system wide install of Spack, but we need a copy that runs from our home directory so we can install/build new packages and perform other actions like creating environments.
Also note that while we are doing this to build hdf5 for openmpi, the instructions below can be modified to build hdf5 for intel-mpi and to run FIESTA with intel-mpi.

## Install user copy of Spack

- `mkdir -p ~/opt`
- `cd ~/opt`
- `git clone https://github.com/spack/spack.git`
- Add to `~/.bashrc`:
  ```
  . ~/opt/spack/share/spack/setup-env.sh
  export PATH=~/opt/spack/bin:$PATH
  ```
 - Restart shell or source new .bashrc (`. ~/.bashrc`)

## Configure spack

We need to tell spack which packages we want so use from the host system so it does not download and build new versions. This both saves time and leverages difficult to build/configure packages that the system administrators have already optimized, like mpi.

- Create `~/.spack/packages.yaml` with the following:
  ```
  packages:
    openmpi:
        externals:
        - spec: openmpi@4.0.5
          modules:
          - openmpi/4.0.5-cuda-77v5
        buildable: False
    intel-mpi:
        externals:
        - spec: intel-mpi@2020.2.254
          modules:
          - intel-mpi/2020.2.254-yvuf
        buildable: False
    cuda:
        externals:
        - spec: cuda@11.2.0
          modules:
          - cuda/11.2.0-w6mf
        buildable: False
    cmake:
        externals:
        - spec: cmake@3.19.5
          modules:
          - cmake/3.19.5-22ub
        buildable: False
    all:
       compiler: [gcc@10.2.0]
       providers:
          mpi: [openmpi]
  ```
- `spack install cmake cuda openmpi`
- `spack install -v hdf5+threadsafe`
- `spack load cmake cuda openmpi hdf5`

Note if you have multiple versions of the same package installed you need to determine which one to use and load. Investigate with `spack find -lv` and `spack graph <hash>`, and then load with something like `spack load /ju6wcsj /vlvnxev /hlsv7m3 /p523iht /lqq6usl`.

## Build FIESTA
  
- `git clone https://github.com/CUP-ECS/fiesta.git`
- `cd fiesta`
- `mkdir build`
- `cd build`
- `cmake .. -DCUDA=on`
- `make -j`

# Run FIESTA

## Interactive session

Assuming 4 nodes

- `salloc --job-name=fiesta-interactive --ntasks=4 --time=02:00:00 --partition=singleGPU --gpus-per-node=1`
- Load dependencies with `module load gcc intel-mpi cuda cmake hdf5` or `spack load cmake cuda openmpi hdf5` for Lmod or Spack, respectively
- `export OMPI_MCA_fs_ufs_lock_algorithm=1`
- Change directory to FIESTA clone
- `cd test/idexp3dterrain/`
- Edit the `--MPI Processors` section of `fiesta.lua` for the number of nodes available such that `procsx * procsy * procsz == num_nodes`.
For this 4 node example we can use `procsx = 2`, `procsy = 2`, and `procsz = 1`.
- `mpirun -n 4 ../../build/fiesta ./fiesta.lua --kokkos-num-devices=1`

## Batch job
- Create a slurm script with contents like below.
Edit paths, module loading, and other values as necessary.
  ```
  #!/bin/bash
  #SBATCH --job-name=fiesta-spack
  #SBATCH --output=./logs/fiesta-spack.%J.out
  #SBATCH --error=./logs/fiesta-spack.%J.err
  #SBATCH --ntasks=4
  #SBATCH --time=00:30:00
  #SBATCH --partition=singleGPU
  #SBATCH --gres=gpu:1

  spack load cmake cuda openmpi hdf5
  export OMPI_MCA_fs_ufs_lock_algorithm=1
  cd ~/programming/fiesta-fork/test/idexp3dterrain/
  mpirun -n 4 ../../build/fiesta ./fiesta.lua --kokkos-num-devices=1
  ```
 - `sbatch <slurm-script>`
