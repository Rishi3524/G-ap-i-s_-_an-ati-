#  Hybrid 2D Convolution with MPI + OpenMP (`conv_stride_mpi_omp.c`)

**Student 1:** Michael Rong Mee Hii — *23237074*  
**Student 2:** Rishwanth Katherapalle — *23463452*  

---

##  Project Description

This program performs **2D convolution with stride** using a **hybrid parallel model** combining **MPI** (distributed memory) and **OpenMP** (shared memory).  
It supports:
- Automatic data decomposition using `MPI_Scatterv` / `MPI_Gatherv`
- Parallel kernel convolution per rank via `#pragma omp parallel for`
- Configurable stride, kernel, and matrix dimensions
- Optional random input generation or reading/writing from text files
- Serial baseline validation for small test matrices

> The implementation follows Assignment 2 specifications and was benchmarked on **Kaya** and **Setonix** HPC systems.

---

##  1. Build Instructions

###  **On Kaya**

**Makefile**
```makefile
CC       = mpicc
CFLAGS   = -O2 -fopenmp -Wall -lm

TARGET   = conv_stride_mpi_omp

all: $(TARGET)

$(TARGET): conv_stride_mpi_omp.c
	$(CC) $(CFLAGS) -o $(TARGET) conv_stride_mpi_omp.c

clean:
	rm -f $(TARGET) *.o
```

**Build command:**
```bash
module load gcc/14.3
module load openmpi/5.0.5
make clean && make
```

---

###  **On Setonix**

**Modules and toolchain:**
Setonix uses the **Cray Programming Environment**.

```bash
module load PrgEnv-gnu
module load cray-mpich
make clean && make
```

---

##  2. Program Usage

Two modes are supported:

### **(a) Random generation**
```bash
./conv_stride_mpi_omp -H H -W W -kH kH -kW kW -sH sH -sW sW [-f f.txt] [-g g.txt] [-o o.txt] [-S SEED]
```

### **(b) File input**
```bash
./conv_stride_mpi_omp -f f.txt -g g.txt -sH sH -sW sW [-o o.txt]
```

### Common Flags:
| Option | Description |
|:--------|:-------------|
| `-H`, `-W` | Input matrix height and width |
| `-kH`, `-kW` | Kernel height and width |
| `-sH`, `-sW` | Stride (vertical, horizontal) |
| `-f`, `-g` | Input matrix and kernel file paths |
| `-o` | Output file path |
| `-S` | Random seed (optional) |

---

##  3. Running on Kaya (UWA HPC)

**Slurm Batch Script:** `run_conv2d.slurm`
```bash
#!/bin/bash
#SBATCH --job-name=conv2d_test
#SBATCH --partition=cits3402
#SBATCH --time=01:00:00
#SBATCH --mem=64G
#SBATCH --nodes=4
#SBATCH --ntasks=4
#SBATCH --cpus-per-task=4

module load gcc/14.3
module load openmpi/5.0.5

make clean
make

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
mpirun -np 4 ./conv_stride_mpi_omp -H 4096 -W 4096 -kH 3 -kW 3 -sH 2 -sW 3
```

**Submit job:**
```bash
sbatch run_conv2d.slurm
```

**Output:**
```
=== Configuration Summary ===
MPI ranks        : 4
OpenMP threads   : 4
Total threads    : 16
==============================
Rank 0 running on node k005.hpc.uwa.edu.au with 4 threads
Comm time (scatter+gather): 0.008s, Compute time: 0.018s
```

---

##  4. Running on Setonix (Pawsey Cray EX)

**Batch Script:** `run_conv2d_setonix.slurm`
```bash
#!/bin/bash
#SBATCH --job-name=conv2d_stress
#SBATCH --account=courses0101
#SBATCH --partition=work
#SBATCH --time=01:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --cpus-per-task=16
#SBATCH --mem=80G
#SBATCH --output=stress_%j.out
#SBATCH --error=stress_%j.err

module load PrgEnv-gnu
module load cray-mpich
module list

make clean
make

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_PLACES=cores
export OMP_PROC_BIND=close

srun ./conv_stride_mpi_omp -H 25000 -W 25000 -kH 15 -kW 15 -sH 1 -sW 1
```

**Submit job:**
```bash
sbatch run_conv2d_setonix.slurm
```

**Example Output:**
```
=== Configuration Summary ===
MPI ranks        : 2
OpenMP threads   : 107
Total threads    : 214
==============================
Rank 0 running on node nid001194 with 107 threads
Rank 1 running on node nid002039 with 107 threads
Dims: A=60000x60000, K=151x151, stride=3x3, out=20000x20000
Comm time (scatter+gather): 2842.802s, Compute time: 45.009s

```

##  5. Output Verification

For small matrices (≤ 256×256 input), the program performs automatic serial validation:

```
Running serial validation (matrix size below threshold)...
Validation (vs serial): max abs diff = 0
```

For large problems, it reports:
```
Skipping serial validation (matrix too large: 60000×60000)
```
