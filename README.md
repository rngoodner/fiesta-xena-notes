# FIESTA Xena notes

Notes on how to build/run [FIESTA](https://github.com/CUP-ECS/fiesta) on [Xena](https://carc.unm.edu/systems/Systems1.html) at [UNM CARC](https://carc.unm.edu).
These notes will cover three methods: 1. by hand, 2. with Spack, and 3. with Singularity.

# Helpful resources

- [CARC HPC documentation](https://github.com/UNM-CARC/QuickBytes/blob/master/README.md)
- [Lmod documentation](https://lmod.readthedocs.io/en/latest/010_user.html)
- [documentation](https://spack.readthedocs.io/en/latest/)
- [Singularity documentation](https://sylabs.io/guides/3.7/user-guide/)

# 1. By-hand

This method uses what is already available on the system.
We will use environment modules (Lmod).

## Build

```
module purge
module load gcc/10.2.0-3kjq intel-mpi/2020.2.254-rxha cuda/11.2.0-w6mf cmake/3.19.5-22ub hdf5/1.10.7-pvyi
git clone https://github.com/CUP-ECS/fiesta.git
cd fiesta
mkdir build
cd build
cmake .. -DCUDA=on
make -j
```

## Slurm batch script

Make sure to read/modify the script before use.
Paths will likely need to be modified.
```
#!/bin/bash
#SBATCH --job-name=fiesta-hand
#SBATCH --output=./logs/fiesta-hand.%J.log
#SBATCH --error=./logs/fiesta-hand.%J.log
#SBATCH --ntasks=8
#SBATCH --time=00:30:00
#SBATCH --partition=singleGPU
#SBATCH --gres=gpu:1

export OMPI_MCA_fs_ufs_lock_algorithm=1
module purge
module load gcc/10.2.0-3kjq intel-mpi/2020.2.254-rxha cuda/11.2.0-w6mf cmake/3.19.5-22ub hdf5/1.10.7-pvyi
cd ~/programming/fiesta-hand/test/idexp3dterrain/
mpirun -n 8 ~/programming/fiesta-hand/build/fiesta ./fiesta.lua --kokkos-num-devices=1
```

# 2. Spack

This will use a local install of Spack so we can compile our own dependencies.

## Install Spack

- `mkdir -p ~/opt`
- `cd ~/opt`
- `git clone https://github.com/spack/spack.git`
- Add to `~/.bashrc`:
  ```
  . ~/opt/spack/share/spack/setup-env.sh
  export PATH=~/opt/spack/bin:$PATH
  ```
 - Restart shell or source new .bashrc (`. ~/.bashrc`)

## Build
```
module purge
spack install gcc@10.2.0
spack load gcc@10.2.0

echo 'spack:
  concretization: together
  specs:
  - cmake
  - cuda
  - openmpi +cuda
  - hdf5 +mpi +threadsafe
  view: true' > ./fiesta-spack.yaml
 
spack env create fiesta-spack fiesta-spack.yaml
spack env activate fiesta-spack
spack install
spack load cmake cuda hdf5+mpi+threadsafe openmpi+cuda
cmake .. -DCUDA=on
make -j
```

## Slurm batch script


Make sure to read/modify the script before use.
Paths will likely need to be modified.
This example uses hashes to uniquely identify packages which may or may not be necessary.
```
#!/bin/bash
#SBATCH --job-name=fiesta-spack
#SBATCH --output=./logs/fiesta-spack.%J.out
#SBATCH --error=./logs/fiesta-spack.%J.err
#SBATCH --ntasks=8
#SBATCH --time=00:30:00
#SBATCH --partition=singleGPU
#SBATCH --gres=gpu:1

export OMPI_MCA_fs_ufs_lock_algorithm=1
module purge
. ~/opt/spack/share/spack/setup-env.sh
~/opt/spack/bin/spack env activate fiesta-spack
~/opt/spack/bin/spack load /sqv6llp /sjettsf /drq3z5z /ulqjmjs # cmake cuda hdf5 openmpi
cd ~/programming/fiesta-spack/test/idexp3dterrain/
mpirun -n 8 ../../build/fiesta ./fiesta.lua --kokkos-num-devices=1
```

## Modifications

Spack is a very powerful tool for managing dependencies and environments.
There are some modifications to the above method that one may want to do.

### packages.yaml

By using a `packages.yaml` file placed in `~/.spack/` one can do many things such as specifying already installed environment modules to use.

Here is an example file that was used successfully on Xena to reduce the number of packages compiled to just 2 packages, specifically hdf5 and zlib.

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

### upstreams.yaml

An `upstreams.yaml` file placed in `~/.spack/` can be used to chain a user's Spack installation to the upstream system Spack installation.
This enables Spack to use all of the system installed Spack packages, while still allowing a non-root user to install new packages as needed.
Note that on Xena it is best to unload all system environment modules to avoid conflicts.
One may also need to compile a new version of gcc, such as 10.2.0, instead of using the one already provided by the system else packages such as perl may fail to build.

Here is an example file that was successfully used on Xena to chain Spack installations.

```
upstreams:
  spack-instance-1:
    install_tree: /opt/spack/opt/spack/
    modules:
      tcl: /opt/spack/share/spack/modules
```

# 3. Singularity

Singularity is a container platform that is common on HPC Clusters.

## Definition file

This definition file uses a nvidia cuda container from dockerhub as a base to build ontop of.
To build a Singularity container one must have root access.
For this test I built the container on a local box and the uploaded it to Xena.
To build a container from a definition file use: `sudo /usr/local/bin/singularity build <output-name>.sif <definition-file>`

```
Bootstrap: docker
From: nvidia/cuda:11.3.0-devel-centos7
Stage: build

%setup

%files
# assumes fiesta cloned into ./fiesta so it can be copied into the container
./fiesta /workspace/fiesta

%environment
export HOME=/workspace

%post
# updates
yum -y update
yum -y groupinstall "Development Tools"

# working dir and new home
mkdir -p /workspace
chmod 777 workspace
cd /workspace
export HOME=/workspace

# spack
git clone https://github.com/spack/spack.git
. $HOME/spack/share/spack/setup-env.sh
spack install gcc@10.2.0 target=x86_64
spack load gcc@10.2.0
spack compiler find

echo 'packages:
  cuda:
    externals:
    - spec: cuda@11.3.0
      prefix: /usr/local/cuda-11.3
    buildable: False
  all:
    compiler: [gcc@10.2.0]
    target: [x84_64]' > $HOME/.spack/packages.yaml

echo 'spack:
  concretization: together
  specs:
  - cmake
  - cuda
  - openmpi +cuda schedulers=slurm
  - hdf5 +mpi +threadsafe
  view: true' > $HOME/fiesta-spack.yaml

# fiesta environment
spack env create fiesta $HOME/fiesta-spack.yaml
spack env activate fiesta
spack install
spack load cmake cuda openmpi hdf5

# fiesta build
cd /workspace/fiesta
rm -rf build
mkdir build
cd build
cmake .. -DCUDA=on -DKokkos_ARCH_KEPLER35=ON
make -j
make install

# cleanup
despacktivate

%runscript
. ~/spack/share/spack/setup-env.sh
spack env activate fiesta
/workspace/fiesta/build/fiesta
```

## Slurm batch script

Make sure to read/modify the script before use.
Paths will likely need to be modified.

```
#!/bin/bash
#SBATCH --job-name=fiesta-singularity
#SBATCH --output=./logs/fiesta-singularity.%J.out
#SBATCH --error=./logs/fiesta-singularity.%J.err
#SBATCH --ntasks=8
#SBATCH --time=00:30:00
#SBATCH --partition=singleGPU
#SBATCH --gres=gpu:1

export OMPI_MCA_fs_ufs_lock_algorithm=1
module purge
module load gcc/10.2.0-3kjq openmpi/4.0.5-yl4z singularity/3.7.0-bm53
cd ~/programming/fiesta-singularity/test/idexp3dterrain/
mpirun -n 8 singularity run --nv ~/fiesta-cuda-slurm.sif ./fiesta.lua --kokkos-num-devices=1
```
