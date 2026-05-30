# Diamond-Dybvig Bank Run Model

A Julia implementation of the classic Diamond-Dybvig (1983) banking model, extended to study how deposit insurance and bank productivity jointly determine bank failure probability across a parameter sweep.

## Overview

The model populates a bank with K identical agents who deposit resources. After an exogenous shock causes some agents to withdraw early, the remaining agents use a Monte Carlo look-ahead to decide endogenously whether to withdraw. If cumulative withdrawals exceed the vault, the bank fails. The code sweeps across three parameters—insurance premium (`insur`), bank productivity (`prod`), and exogenous withdrawal probability (`objP`)—and records the failure rate for each combination.

## File Structure

| File | Purpose |
|------|---------|
| `objects.jl` | Struct definitions: `Agent`, `Bank`, `Model`, `SimModel` |
| `functions.jl` | Core logic: utility, single-round simulation, withdrawal, payout |
| `main0001.jl` | Main entry point — distributed parameter sweep, writes `results.csv` |
| `convergence_test.jl` | Verifies stochastic dominance across K=50, 100, 200 |
| `smallEps.jl` | Sweeps small insurance values (0.0–0.1) for bridge-theorem regime |
| `time_k50.jl` / `time_k500.jl` | Timing benchmarks for K=50 and K=500 |
| `plotGen.jl` / `plotGen2.jl` | Julia-side visualization of agent and bank dynamics |
| `parzenTesting.jl` | TreeParzen hyperparameter optimization experiments |
| `analysis.R` | R script — generates PDF plots from results CSVs |
| `container.def` | Singularity container definition (Julia 1.11.4) |

## Dependencies

### Julia (1.11+)

Install packages once:

```julia
using Pkg
Pkg.add(["Distributions", "Random", "CSV", "DataFrames",
         "Graphs", "StatsBase", "JLD2", "Dates",
         "TreeParzen", "Plots", "Folds"])
```

### R (for analysis)

```r
install.packages(c("tidyverse", "ggplot2", "data.table"))
```

## How to Run

### 1. Main Parameter Sweep

```bash
julia main0001.jl
```

This spawns workers across all available CPU cores and runs a grid over:

- `insur` ∈ {0.5, 0.6}
- `prod` ∈ {0.5, 0.6}
- `objP` ∈ 51 evenly-spaced values from 0.0 to 1.0

Results are written to `../DDData/results.csv` (create that directory first):

```bash
mkdir -p ../DDData
julia main0001.jl
```

### 2. Convergence Test

Checks that failure probability decreases monotonically as K grows (stochastic dominance). Runs 15 representative parameter tuples at K = 50, 100, 200. Estimated runtime: ~3.5 hours.

```bash
mkdir -p ../DDData
julia convergence_test.jl
```

Output: `../DDData/convergence_results.csv`

### 3. Small-Epsilon Sweep

Sweeps insurance values in [0.0, 0.1] to populate the low-insurance regime.

```bash
julia smallEps.jl
```

Output: `../DDData/results_smallEps.csv`

### 4. Analysis and Plots

After collecting results:

```bash
Rscript analysis.R
```

Generates `results_{prod_value}.pdf` showing failure probability as a function of insurance and withdrawal probability.

### 5. Container Execution (Linux/HPC)

```bash
singularity run container.def
```

Runs `main0001.jl` inside a Julia 1.11.4 Singularity container.

## Key Parameters

| Parameter | Description |
|-----------|-------------|
| `insur` | Deposit insurance premium (fraction of deposit returned on failure) |
| `prod` | Bank productivity (return on invested deposits) |
| `objP` | Exogenous probability an agent withdraws early |
| `agtCnt` (K) | Number of agents |
| `depth` | Monte Carlo simulation depth for agent look-ahead |
| `riskAversion` | CRRA risk aversion coefficient (log utility when = 1) |
| `totResr` | Total resources available per agent |

## Output Schema

`results.csv` columns:

| Column | Description |
|--------|-------------|
| `insur` | Insurance level |
| `prod` | Productivity |
| `objP` | Exogenous withdrawal probability |
| `failProb` | Fraction of simulation runs ending in bank failure |
