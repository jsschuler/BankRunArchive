# Network Agent-Based Bank Run Model

A Julia agent-based model (ABM) studying how bank runs cascade through social networks. Agents observe their neighbors' withdrawal behavior to form expectations about systemic risk, then decide endogenously whether to withdraw. The model sweeps over bank reserve ratios, deposit insurance levels, deposit distributions, and network topologies.

This code accompanies the paper *"A Bank Run Model for the Twentieth Century"* (John S. Schuler, George Mason University, 2025).

## Overview

One thousand agents hold deposits drawn from a parametric distribution (LogNormal or Pareto) and are connected via a Watts-Strogatz small-world network. An initial exogenous shock (geometric random variable) forces some agents to withdraw. Each remaining agent then runs an internal Monte Carlo simulation—cloning the current bank state and sampling possible futures—to compare the loss probability of withdrawing now versus waiting. Agents withdraw if the immediate risk exceeds the expected future risk. The bank fails when its vault reaches zero.

## File Structure

| File | Purpose |
|------|---------|
| `objects.jl` | Struct definitions: `Agent`, `Bank`, `SimModel`, network wrappers |
| `functions4.jl` | Core model logic: agent decisions, withdrawals, Monte Carlo sub-simulations |
| `parameterGen.jl` | Generates the full parameter grid |
| `finMain0001.jl` | Main entry point — distributes work across 16 Julia workers |
| `modelStep.jl` | Executes a single model step (one parameter row) |
| `dataTools.jl` | CSV aggregation and data utilities |
| `restart.jl` | Resumes an interrupted sweep from the last completed row |
| `jld2CSV.jl` | Converts JLD2 binary output to CSV |
| `run_sweep.sh` | Bash script to launch the full parameter sweep |
| `sysImage.jl` | Builds a Julia sysimage to speed up worker startup |
| `analysis.R` | Basic post-run analysis |
| `analysis2.R` | Comprehensive visualizations (histograms, scatterplots, summaries) |
| `finAnalysis.R` | Additional financial analysis |
| `workingAnalysis.R` | Exploratory analysis notebook |
| `draft.Rnw` / `draft.pdf` | Sweave source and compiled paper |

## Dependencies

### Julia (1.11+)

```julia
using Pkg
Pkg.add(["Distributions", "Random", "CSV", "DataFrames",
         "Graphs", "StatsBase", "JLD2", "Dates"])
```

Optionally build a sysimage for faster startup on HPC clusters:

```bash
julia sysImage.jl
```

### R (for analysis)

```r
install.packages(c("tidyverse", "data.table", "ggExtra"))
```

## How to Run

### 1. Single Run (manual parameters)

```bash
julia finMain0001.jl \
  <DATA_DIR> <GEN_SEED> <RESERVE_RATIO> <DEP_QUANTILE> \
  <DIST_NAME> <LOGNORM_MU> <LOGNORM_SIGMA> <WS_K> <WS_P>
```

Example:

```bash
mkdir -p ../BankRunData
julia finMain0001.jl ../BankRunData 42 0.2 0.1 LogNormal 0.0 2.0 6 0.05
```

#### Arguments

| Argument | Description | Example values |
|----------|-------------|----------------|
| `DATA_DIR` | Output directory | `../BankRunData` |
| `GEN_SEED` | Random seed | `42` |
| `RESERVE_RATIO` | Bank reserve ratio | `0.2`, `0.4` |
| `DEP_QUANTILE` | Deposit insurance quantile | `0.1`, `0.2`, `0.3`, `0.4` |
| `DIST_NAME` | Deposit distribution | `LogNormal`, `Pareto` |
| `LOGNORM_MU` | LogNormal μ | `0.0` |
| `LOGNORM_SIGMA` | LogNormal σ | `2.0`, `3.0` |
| `WS_K` | Watts-Strogatz avg degree | `6`, `10`, `50` |
| `WS_P` | Watts-Strogatz rewiring prob | `0.05`, `0.15` |

### 2. Full Parameter Sweep

```bash
chmod +x run_sweep.sh
./run_sweep.sh
```

Sweeps all combinations of:
- Reserve ratios: {0.2, 0.4}
- Insurance quantiles: {0.1, 0.2, 0.3, 0.4}
- LogNormal σ: {2.0, 3.0}
- WS k: {6, 10, 50}
- WS p: {0.05, 0.15}

### 3. Resuming an Interrupted Sweep

```bash
julia restart.jl <DATA_DIR>
```

Reads completed results and re-queues only the missing parameter rows.

### 4. Analysis and Visualization

After results are collected in `DATA_DIR`:

```bash
Rscript analysis2.R <DATA_DIR>
```

Generates:
- Histograms of endogenous/exogenous withdrawals by failure outcome
- Scatterplots of withdrawal probabilities with marginal distributions
- Summary tables: failure rate by reserve ratio, network structure, insurance level

## Output Files

Each run writes to `DATA_DIR`:

| File | Contents |
|------|---------|
| `bankRunParametersInit.csv` | Full parameter grid |
| `bankRunResults<N>.csv` | Bank failure outcome (boolean) per run, per worker |
| `bankRunEndogenous<N>.csv` | Per-agent endogenous decisions, probabilities, timing |
| `bankRunExogenous<N>.csv` | Exogenous shock withdrawals |
| `agents<N>.csv` | Initial agent deposit amounts |

`<N>` is the worker process ID.

## Key Model Parameters

| Parameter | Description |
|-----------|-------------|
| Reserve ratio | Fraction of deposits held as liquid reserves |
| Insurance quantile | Deposits below this quantile are fully insured |
| WS k | Average number of network neighbors per agent |
| WS p | Probability of edge rewiring (controls "small-worldness") |
| Deposit distribution | Shape of wealth heterogeneity across agents |
| Monte Carlo depth | Number of sub-simulations per agent decision (default: 1000) |
| Exogenous shock | Geometric(p=0.1) number of forced early withdrawals |
