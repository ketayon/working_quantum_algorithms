# working_quantum_algorithms

 Small Jupyter notebooks that implement and compare flagship quantum algorithms:

- `grover_search_secret_number.ipynb` – Grover search against a classical linear scan.
- `QAOA_Max_Cut.ipynb` – QAOA for Max-Cut on ring graphs with hardware sampling.
- `monte_carlo_IQAE _MLAE.ipynb` – Classical Monte Carlo vs IQAE/MLAE for amplitude estimation.
- `Schur_Grover.ipynb` – Grover search for Schur-number toy colorings.

### Shared workflow patterns
- **Always keep a classical baseline in the loop.** Every notebook defines a classical solver (`classical_linear_search`, `brute_force_maxcut`, `classical_monte_carlo`, `find_schur_valid_coloring`) and times it, so you can quantify algorithmic speedups and validate quantum outputs.
- **Modularize circuit building blocks.** Oracles, diffusers, ansatz layers, amplitude-preparation circuits, and Grover subroutines are built as separate functions (for example `build_oracle`, `build_diffuser`, `qaoa_ansatz`, `build_amplitude_circuit`). This keeps tests focused and lets you reuse compiled modules across simulators and hardware.
- **Transpile once before execution.** All runs go through `generate_preset_pass_manager(..., optimization_level=1)` to produce ISA-level circuits that respect backend constraints. Reuse the transpiled circuit for both Aer simulators and IBM hardware runs to maintain parity.
- **Treat IBM Runtime access as optional.** The shared `get_runtime_service` / `get_backend` helpers save the API token when provided, pick the least busy device, and gracefully fall back to `AerSimulator` when hardware is unavailable. Simulate first, then enable `use_ibm=True` to submit the exact same circuit to real hardware.
- **Expose interactive drivers.** Functions such as `grover_interactive`, `qaoa_interactive`, `qae_modern_interactive`, and `schur_grover_interactive` wrap the primitives above so you can sweep parameters (number of qubits, shots, optimizer depth, error budgets) without editing the core logic.

### Notebook-specific highlights
- **Grover search (`grover_search_secret_number.ipynb`).**  
  - `grover_iterations` enforces the π/4√N iteration heuristic, preventing over-rotation.  
  - `run_grover_quantum` transpiles the circuit once, measures backend runtime, and returns raw counts so you can plot and reason about success probability. Always compare against `classical_linear_search` to sanity-check the secret that Grover retrieves.

- **QAOA Max-Cut (`QAOA_Max_Cut.ipynb`).**  
  - The ansatz is parameterized with `Parameter` objects so COBYLA can reuse a pre-transpiled circuit during optimization.  
  - `estimate_energy_by_sampling` computes energies from sampled bitstrings instead of the Estimator primitive, which keeps the notebook stable across Qiskit versions and surfaces shot noise explicitly.  
  - Run the full optimizer on Aer first (`run_qaoa_on_simulator`) and only submit the optimal bound circuit to IBM via `sample_on_ibm`; this mirrors how near-term QAOA studies de-risk hardware shots.

- **Modern amplitude estimation (`monte_carlo_IQAE _MLAE.ipynb`).**  
  - `build_estimation_problem` isolates state preparation so IQAE/MLAE share identical problem definitions.  
  - The IQAE helpers expose `(epsilon, alpha, shots)` clearly; tighten or loosen them before touching hardware to control oracle-call budgets.  
  - `run_mlae_sim_manual` reconstructs the MLAE likelihood curve from simulator counts, which is a good diagnostic before trusting `MaximumLikelihoodAmplitudeEstimation` on noisy devices.  
  - Hardware runners (`run_iqae_ibm`, `run_mlae_ibm`) lean on `RuntimeSampler` and reuse the same Grover powers, keeping simulator vs. hardware comparisons apples-to-apples.

- **Schur Grover (`Schur_Grover.ipynb`).**  
  - Always solve the toy instance classically first (`find_schur_valid_coloring`) to obtain a known-valid coloring that Grover can target; encode it with `coloring_to_int`/`int_to_bitstring` so the oracle is self-consistent.  
  - `build_grover_schur_toy_circuit` separates oracle/diffuser assembly and reports the chosen iteration count; plot histograms from both simulator and IBM runs to see how success probability tracks the theoretical prediction.  
  - `run_grover_schur_toy` closes with runtime and step comparisons (2ⁿ classical checks vs. ≈π/4√(2ⁿ) Grover iterations), reinforcing why the iterative scaling analysis matters.

## Results & conclusions (current notebook runs)
- **Grover search:** For the 4-qubit secret `9 (1001)`, the classical linear scan needed 10 steps (≤16 worst-case) and finished in about 2 microseconds. Grover completed 3 iterations, achieved ~0.96 success probability on Aer in 2.8 ms, and ~0.46 on IBM `ibm_fez` in 5.95 s—same logical circuit, but hardware noise lowers the peak.
- **QAOA Max-Cut:** On a 10-vertex ring graph the classical brute-force solver found the optimum cut value 10 after checking all 1024 assignments in about 1.4 ms. Sampling-based QAOA optimization on Aer converged in roughly 20 iterations to an average cut of ~8.08 (optimization 0.27 s, sampling 4.6 ms). The same optimized circuit on IBM `ibm_fez` produced an average cut ~6.80 with 5.6 s sampling time, highlighting that hardware still beats classical in evaluation count (roughly 50× fewer cost evaluations) even if wall-clock is slower.
- **Modern amplitude estimation:** With true probability `p = 0.3`, classical Monte Carlo using 10,000 samples estimated 0.3029 in 0.3 ms. The simulator-based quantum sampler with the same shot budget estimated 0.3035 (7.7 ms). Manual single-qubit MLAE on Aer using ks = (0,1,2,3) and 1000 shots each achieved 0.2991 in 0.25 s (about 16,000 effective oracle calls). On IBM hardware the naive quantum sampler hit 0.3013 in 9.7 s; MLAE reached 0.3029 in 8.3 s. IQAE targeted ε = 0.02, requiring only about 50 effective Grover queries (O(1/ε) scaling versus the classical O(1/ε^2) sample complexity).
- **Schur Grover:** For the n = 4, k = 2 toy coloring, the classical brute force checked 7 of the 16 configurations before finding a valid `[0,1,1,0]` assignment. Grover required ~3 iterations, yielding ~0.96 success probability on Aer in 2.3 ms and ~0.43 on IBM `ibm_torino` in 5.1 s. The experiment confirms the quadratic reduction from O(2^n) classical checks to O(2^{n/2}) Grover iterations while illustrating the current hardware noise floor.

