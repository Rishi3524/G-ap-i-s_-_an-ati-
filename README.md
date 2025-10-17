# conv_stride_mpi_omp (MPI + OpenMP Hybrid) — Running on **Kaya** and **Setonix** (UWA / Pawsey HPC)

**Student 1:** Michael Rong Mee Hii — 23237074  
**Student 2:** Rishwanth Katherapalle — 23463452  

This README shows how to build and run the hybrid 2-D convolution program (`conv_stride_mpi_omp.c`) on both the **Kaya** (UWA HPC) and **Setonix** (Pawsey Cray) clusters using **Slurm**.

> The program implements a hybrid MPI + OpenMP 2-D convolution with stride, automatic row decomposition using `MPI_Scatterv` / `MPI_Gatherv`, and OpenMP parallel loops for local kernel computation.

---

## 1) Prerequisites

### For **Kaya**
- You have a **Kaya** account and can SSH into the login node:
  ```bash
  ssh <username>@kaya01.hpc.uwa.edu.au
  ```
- Use **Slurm** to run compute jobs (do not run on login node).
- Recommended working directories:
  - `/group/cits3402/<username>` — shared project space
  - `/scratch/cits3402/<username>` — fast temporary storage

### For **Setonix**
- You have a Pawsey account and project allocation (e.g. `courses0101`).
- Access via:
  ```bash
  ssh <username>@setonix.pawsey.org.au
  ```

---

## 2) Getting the code onto the cluster

From your local machine:
```bash
scp -r <local_project_dir> <username>@kaya01.hpc.uwa.edu.au:/group/cits3402/<username>/conv2d
# or
scp -r <local_project_dir> <username>@setonix.pawsey.org.au:/scratch/courses0101/<username>/conv2d
```

Then:
```bash
ssh <username>@kaya01.hpc.uwa.edu.au
cd /group/cits3402/<username>/conv2d
```
(and the equivalent Setonix path)

---

## 3) Build

Place a **Makefile** at the root of the project.

### For Kaya
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

**Build commands:**
```bash
module load gcc/14.3
module load openmpi/5.0.5
make clean && make
```

---

### For Setonix
```bash
module load PrgEnv-gnu
module load cray-mpich
make clean && make
```

---

## 4) Run via Slurm (Batch Script)

### On **Kaya**
Create `run_conv2d.slurm`:
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

Submit:
```bash
sbatch run_conv2d.slurm
```

---

### On **Setonix**
Create `run_conv2d_setonix.slurm`:
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

Submit:
```bash
sbatch run_conv2d_setonix.slurm
```

---

## 5) Program Usage

The executable `conv_stride_mpi_omp` supports **two modes**:

- **Generate mode** — create random input/kernel and optionally save:
  ```bash
  ./conv_stride_mpi_omp -H H -W W -kH kH -kW kW -sH sH -sW sW [-f f.txt] [-g g.txt] [-o o.txt] [-S SEED]
  ```

- **Read mode** — use existing files:
  ```bash
  ./conv_stride_mpi_omp -f f.txt -g g.txt -sH sH -sW sW [-o o.txt]
  ```

**Common options:**

| Flag | Description |
|:------|:-------------|
| `-H`, `-W` | Input matrix size |
| `-kH`, `-kW` | Kernel size |
| `-sH`, `-sW` | Stride (vertical, horizontal) |
| `-f`, `-g` | Input matrix / kernel file paths |
| `-o` | Output file path |
| `-S` | Random seed (optional) |

---

## 6) Example Runs

### Kaya (UWA HPC)
```bash
# 4 MPI × 4 OpenMP
mpirun -np 4 ./conv_stride_mpi_omp -H 4096 -W 4096 -kH 3 -kW 3 -sH 2 -sW 3
```

Example output:
```
=== Configuration Summary ===
MPI ranks        : 4
OpenMP threads   : 4
Total threads    : 16
==============================
Comm time (scatter+gather): 0.008s, Compute time: 0.018s
```

---

### Setonix (Pawsey Cray)
```bash
srun ./conv_stride_mpi_omp -H 60000 -W 60000 -kH 151 -kW 151 -sH 3 -sW 3
```

Example output:
```
=== Configuration Summary ===
MPI ranks        : 2
OpenMP threads   : 107
Total threads    : 214
==============================
Dims: A=60000x60000, K=151x151, stride=3x3, out=20000x20000
Comm time (scatter+gather): 2842.802s, Compute time: 45.009s
```

---

## 7) Validation and Cleanup

For small matrices (≤ 256×256):
```
Running serial validation (matrix size below threshold)...
Validation (vs serial): max abs diff = 0
```

For large problems:
```
Skipping serial validation (matrix too large: 60000×60000)
```
