# Quantum ESPRESSO Installation Guide (GPU-Enabled)

This guide provides step-by-step instructions for compiling and installing **Quantum ESPRESSO (QE) with GPU acceleration** using the **NVIDIA HPC SDK** and **OpenMPI**. It also includes common troubleshooting tips and how to determine system architecture.

---

## Why use GPUs?
They can significantly accelerate calculations that can often take days and or weeks!

This installation guide is largely adapted from Simon Gelin's README's found in many software folders. 

## Installing Quantum Espresso - GPU on Pittsburgh Super Computer ([PSC-Bridges2](https://www.psc.edu/resources/bridges-2/))

### PSC GPU architecture

Installation was carried out with the following modules loaded. It is important to check what capabilities you would like to enable for the installation and your system. That will then determine what modules would be required. 

```
$ module purge
$ module load allocations/1.0 psc.allocations.user/1.0 nvhpc/22.9 openmpi/4.0.5-nvhpc22.9 cuda/11.7.1 fftw
$ source
```


**1. Launch an interactive session on a GPU node.**
It is important to compile on a GPU node.

`$ interact --gpu -t 8:00:00`






**2. Checking System Architecture**

Once the session is loaded, check your details of the machine using 
`$ nvaccelinfo`

```console
CUDA Driver Version:           12060
NVRM version:                  NVIDIA UNIX x86_64 Kernel Module  560.35.03  Fri Aug 16 21:39:15 UTC 2024

Device Number:                 0
Device Name:                   Tesla V100-SXM2-32GB
Device Revision Number:        7.0
Global Memory Size:            34072559616
Number of Multiprocessors:     80
Concurrent Copy and Execution: Yes
Total Constant Memory:         65536
Total Shared Memory per Block: 49152
Registers per Block:           65536
Warp Size:                     32
Maximum Threads per Block:     1024
Maximum Block Dimensions:      1024, 1024, 64
Maximum Grid Dimensions:       2147483647 x 65535 x 65535
Maximum Memory Pitch:          2147483647B
Texture Alignment:             512B
Clock Rate:                    1530 MHz
Execution Timeout:             No
Integrated Device:             No
Can Map Host Memory:           Yes
Compute Mode:                  default
Concurrent Kernels:            Yes
ECC Enabled:                   Yes
Memory Clock Rate:             877 MHz
Memory Bus Width:              4096 bits
L2 Cache Size:                 6291456 bytes
Max Threads Per SMP:           2048
Async Engines:                 6
Unified Addressing:            Yes
Managed Memory:                Yes
Concurrent Managed Memory:     Yes
Preemption Supported:          Yes
Cooperative Launch:            Yes
  Multi-Device:                Yes
Default Target:                cc70
```

Use to get the specific information on the target and driver use: 
`$ nvaccelinfo | grep -e 'Target' -e 'Driver'`

![Output from nvaccelinfo](https://github.com/MosaicGroupCMU/Softwares/blob/c8040188aee443a7a328b145b9e498feacc35bbb/Quantum-Espresso-GPU/Images/Picture2.png "a title")

You could also use the following command to find more details on the gpu node and which CUDA Toolkit is being used: 

`$ nvidia-smi`

![output from nvidia-smi](https://github.com/MosaicGroupCMU/Softwares/blob/c8040188aee443a7a328b145b9e498feacc35bbb/Quantum-Espresso-GPU/Images/Picture1.png "a title")

Similarly, check the version of Nvidia C compiler and mpirun currently using. 
```
nvcc --version
mpirun --version
```

**3. Get the current package of QE from github** 
```console
$ cd /path/where/you/want/qe/to/be/installed/
$ mkdir qe
$ cd qe
$ git clone https://gitlab.com/QEF/q-e.git qe-7.3.1
$ cd ./qe-7.3.1
$ git fetch --all --tags
$ git checkout tags/qe-7.3.1 -b qe-7.3.1
``` 
**4. Configure your installation using CMake**
This is the CMAKE command I used to compile. You can customize it as you wish to include Environ or LibXC. Ideally, you would specify only what is necessary and what your system cannot find for itself.
```console
cmake -DCMAKE_C_COMPILER=nvc
-DCMAKE_Fortran_COMPILER=nvfortran
-DQE_ENABLE_PROFILE_NVTX=ON
-DQE_ENABLE_CUDA=ON
-DQE_ENABLE_OPENACC=ON
-DQE_ENABLE_OPENMP=OFF
-DNVFORTRAN_CUDA_CC=70
-DNVFORTRAN_CUDA_VERSION=11.7
-DQE_ENABLE_MPI_GPU_AWARE=ON
-DCUDA_LIBS=cufft,cublas,cusolver,curand ..
```
As one line: 
```
cmake -DCMAKE_C_COMPILER=nvc -DCMAKE_Fortran_COMPILER=nvfortran -DQE_ENABLE_PROFILE_NVTX=ON -DQE_ENABLE_CUDA=ON -DQE_ENABLE_OPENACC=ON -DQE_ENABLE_OPENMP=OFF -DNVFORTRAN_CUDA_CC=70 -DNVFORTRAN_CUDA_VERSION=11.7 -DQE_ENABLE_MPI_GPU_AWARE=ON -DCUDA_LIBS=cufft,cublas,cusolver,curand ..
```
**5. Build your configured system**
Finally, make the software. The following command compiles the code using 16 processors and puts all of the process outputs into the "build.log".

```
make -j16 > build.log 2>&1
```


## Check your installation

Test input file: 

```fortran
&control
    calculation = 'scf'
    prefix = 'silicon'
    pseudo_dir = '/path/to/pseudopotential'
    outdir = './tmp'
/
&system
    ibrav = 2,
    celldm(1) = 10.2,
    nat = 2,
    ntyp = 1,
    ecutwfc = 30.0,
    ecutrho = 240.0,
/
&electrons
    conv_thr = 1.0d-8
    mixing_beta = 0.7
/
ATOMIC_SPECIES
Si  28.0855  Si.upf
ATOMIC_POSITIONS crystal
Si  0.00  0.00  0.00
Si  0.25  0.25  0.25
K_POINTS automatic
8 8 8 1 1 1
```


Here is a sample submission script to test. 

```bash
#!/bin/bash
#SBATCH -N 1
#SBATCH -p GPU-shared
#SBATCH -t 0:10:00
#SBATCH -n 4
#SBATCH --gpus=v100-16:1
#SBATCH -o slurm_output.out      
#SBATCH -e slurm_error.err

#type 'man sbatch' for more information and options
#this job will ask for 1 V100 GPUs on a v100-16 node in GPU-shared for 10 minutes 


# move to working directory
# this job assumes:
# - all input data is stored in this directory
# - all output should be stored in this directory
# - please note that groupname should be replaced by your groupname
# - PSC-username should be replaced by your PSC username
# - path-to-directory should be replaced by the path to your directory where the executable is

cd /jet/home/chandlec/group-projects/chandlec/test

#Helps the cpus organize themselves
export OMP_NUM_THREADS=4

#run pre-compiled program which is already in your project space

/jet/home/chandlec/group-projects/chandlec/software/qe/qe-7.3.1/build/bin/pw.x < input.in > output.out
```

A successful installation of GPU-enabled QE will say the following in the output file. 

![showing where it says gpu-accellerated within the beginning of QE output](https://github.com/MosaicGroupCMU/Softwares/blob/c8040188aee443a7a328b145b9e498feacc35bbb/Quantum-Espresso-GPU/Images/Picture3.png) "a title")
![Table within QE output showing the amount of calls to GPUs](https://github.com/MosaicGroupCMU/Softwares/blob/c8040188aee443a7a328b145b9e498feacc35bbb/Quantum-Espresso-GPU/Images/Picture4.png) "a title")

# Additional Resources

* QE CMAKE [Instructions](https://gitlab.com/QEF/q-e/-/wikis/Developers/CMake-build-system)
* Quantum Espresso Mailing List [here](https://lists.quantum-espresso.org/pipermail/users/2023-July/050687.html#:~:text=url=https%253A%252F,~/.bashrc%20*Robinson%20J.)
* MaX Webinar on ["How to use Quantum ESPRESSO on new GPU based HPC systems"](https://www.youtube.com/watch?v=76xH_mpw8CU)
* Quantum ESPRESSO & GPU survival Guide [Presentation](https://docs.epw-code.org/_downloads/e9a4fa9f61302bb55b5f804bcc805880/Sat.1.Giannozzi.pdf)
* Quantum ESPRESSO [User Guide](https://www.quantum-espresso.org/Doc/user_guide_PDF/user_guide.pdf)
* Another installation [guide](https://www.youtube.com/watch?v=Z3-dkOpfEqE) and benchmarking too!
* Quantum ESPRESSO on HPC and GPU systems:parallelization and hybrid architectures [presentation](https://indico.ictp.it/event/9616/session/53/contribution/89/material/slides/0.pdf)
* [Accelerate Quantum Espresso simulations with GPU Shapes on OCI](https://blogs.oracle.com/cloud-infrastructure/post/accelerate-quantum-espresso-simulation-oci-gpu)
* QUANTUM ESPRESSO on HPC systems: [Hands-on-session](https://enccs.github.io/max-coe-workshop/_downloads/742ab11aebbb2123596209ded0db2c0b/Handson-Day1.pdf)

# Troubleshooting
Come back later for information on this!

* Nvidia does not support fortran 2008! Will cause errors with HDF5 as seen [here](https://docs.nersc.gov/systems/perlmutter/vendorbugs/#missing-mpi_f08-module)
* Problems installing Quantum epsresso with GPU acceleration [here](https://forums.developer.nvidia.com/t/problems-installing-quantum-epsresso-with-gpu-acceleration/189908)
* Compiling Quantum ESPRESSO with GPU support [here](https://forums.developer.nvidia.com/t/compiling-quantum-espresso-with-gpu-support/222227)

