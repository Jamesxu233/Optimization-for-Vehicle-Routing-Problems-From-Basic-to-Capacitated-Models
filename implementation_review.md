# Implementation Review: VRP Optimization Notebook and Report

This document records alignment checks between the supplied `Project Report.pdf`, `README(1).md`, and `code(1).ipynb`. It distinguishes the report's claims from what is visibly implemented in the notebook.

## 1. Notebook structure

The notebook contains eight executable solver sections:

| Section | Notebook cell | Implementations |
|---|---:|---|
| TSP | 0–1 | Google OR-Tools |
| TSP | 2–3 | Simulated Annealing |
| TSP | 4–5 | Genetic Algorithm |
| Four-vehicle VRP | 6–7 | Simulated Annealing |
| Four-vehicle VRP | 8–9 | Genetic Algorithm |
| Four-vehicle VRP | 10–11 | Google OR-Tools |
| CVRP | 12–13 | Genetic Algorithm |
| CVRP | 14–15 | Google OR-Tools |

The notebook does **not** include a separate CVRP Simulated Annealing cell, although the report's Table III lists a CVRP Simulated Annealing result.

## 2. Results cross-check

| Problem | Report | Saved notebook output | Review |
|---|---|---|---|
| TSP SA | 4,544 | 4,544 | Consistent |
| TSP GA | 4,544 | 4,544 | Consistent |
| TSP OR-Tools | 4,384 | 4,384 | Consistent |
| 4-vehicle VRP SA | 8,536 | 8,536 | Consistent |
| 4-vehicle VRP GA | 7,372 | 7,372 | Consistent |
| 4-vehicle VRP OR-Tools | 6,300 route-distance sum | Four route distances sum to 6,300; printed objective is 177,500 | Explain objective-vs-distance distinction |
| CVRP GA | 6,208 | 6,208 | Consistent for saved run |
| CVRP SA | 6,208 | No implementation/output cell | Not independently verifiable from supplied notebook |
| CVRP OR-Tools | 6,208 | 6,208 | Consistent for saved run |

### OR-Tools 4-vehicle VRP objective

The 4VRP OR-Tools cell adds a `Distance` dimension with maximum route distance 3,000 and sets `SetGlobalSpanCostCoefficient(100)`. The printed `Objective: 177500` therefore includes the global-span term. The four route-distance values printed by the cell are 1,712, 1,484, 1,552, and 1,552, which sum to 6,300. The report's 6,300 is the total route distance; it should not be compared directly with the solver's augmented objective value.

## 3. CVRP modeling details

The notebook uses:

- 17 nodes (depot 0 and 16 customers)
- 4 vehicles
- Capacity 15 for each vehicle
- Demand vector `[0, 1, 1, 2, 4, 2, 4, 8, 8, 1, 2, 1, 2, 4, 4, 8, 8]`
- Total customer demand 60
- Total fleet capacity 60

The saved OR-Tools output reports a load of 15 on each vehicle and a total route distance of 6,208.

## 4. Recommended code validation

### 4.1 Validate every CVRP candidate after repair

The CVRP GA's `repair_individual` routine attempts to remove duplicates, respect capacity, and insert missing nodes. However, if no route has remaining capacity, a missing node can remain unassigned. The function does not raise an error or explicitly verify complete customer coverage at the end.

Add a validator that checks:

- Every customer node `1..16` is visited exactly once.
- No customer is omitted or duplicated.
- Every route starts and ends at depot `0`.
- Every route's demand sum is at most 15.
- No route contains invalid node indices.
- The number of routes does not exceed four.

Reject or repair infeasible candidates before fitness evaluation.

### 4.2 Reconcile CVRP GA tournament selection

The CVRP notebook declares `tournament_size = 50`, but the selection function uses a hard-coded sample size of `250` in `np.random.randint`. The population size is 10. Since indices are sampled with replacement, this is not a conventional 50-individual tournament and does not match the declared parameter.

Use the `tournament_size` variable consistently, and constrain it to a meaningful value relative to the population size. Prefer sampling without replacement where appropriate.

### 4.3 Check route mutation edge cases

The CVRP mutation samples two interior customer positions from a route. Routes with fewer than two interior customers can raise a sampling error. Add a length guard before attempting swap mutation.

### 4.4 Make random runs reproducible

The notebook uses both `numpy.random` and Python's `random` module. Set both seeds (and record them) for each experiment. For robust reporting, run multiple seeds and summarize mean, standard deviation, best cost, and runtime instead of relying on one saved stochastic run.

### 4.5 Validate initial solutions

The CVRP initialization functions stop after the configured number of vehicles. Depending on the random assignment and capacity constraints, customers may remain unassigned. Validate the initial population before starting evolution, and use a deterministic fallback or a constructive feasibility procedure.

### 4.6 Compare like-for-like objective values

The 4VRP OR-Tools cell uses a global-span cost coefficient. Its `ObjectiveValue()` includes more than the sum of travel distances. Keep the route-distance sum and maximum route distance as separate metrics. Compare total travel distance to the report's 6,300.

## 5. Report and code alignment

- The report describes SA and GA for TSP and four-vehicle VRP, and a GA-based CVRP extension.
- The report's CVRP results table includes SA and OR-Tools, but the uploaded notebook has only CVRP GA and OR-Tools implementations.
- The notebook's CVRP GA uses population size 10, 50 generations, mutation rate 0.05, vehicle capacity 15, and a declared tournament size 50. The selection function currently hard-codes 250.
- The notebook's TSP and 4VRP GA use population size 100, 500 generations, tournament size 5, and mutation rate 0.05.
- The notebook's 4VRP SA sets initial temperature 100,000, cooling rate 0.95, and 10,000 iterations. The report's 4VRP section says the parameters are the same as the previous implementation; this should be reconciled with the notebook.
- The report's 4VRP OR-Tools table presents 6,300 as the optimal value. The notebook's printed route-distance sum is 6,300, while its augmented objective is 177,500.

## 6. Suggested maintenance order

1. Replace the repository README with a structured project overview and exact problem progression.
2. Keep the report PDF as a reference artifact and preserve the notebook's original experimental sections.
3. Add the objective-vs-route-distance explanation for 4VRP OR-Tools.
4. Add CVRP solution-feasibility validation and fix tournament-size handling.
5. Add deterministic seeds and repeated-run statistics.
6. Refactor the notebook into modules only after the original results and behavior are reproducibly validated.
