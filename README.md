# Verifiable Quantum Circuit Compilation: Google Quantum AI

Cheaper quantum circuits, checked by a machine that tests every answer.

> **Status (7 Nov 2026):** Work in progress. `invsqrt` and `dqi` are improved and pass the official verifiers. `dfthcsos` is not started (the untouched sample is used).
> **Leaderboard score:** _to be added after the first submission is scored_

---

## Results

| Task | Sample (reference) | This work | Improvement | Verified with |
|---|---|---|---|---|
| **invsqrt** (fusion energy) | 2,076,774 (260 qubits, 7,735 Toffolis) | **555,080** (164 qubits, 3,139 Toffolis) | **3.74x** | Official verifier, all 200,540,928 shots passed |
| **dqi** (optimization) | ~2.44e13 (1,893 qubits, 5,725,461 Toffolis) | **2.11e13** (1,893 qubits, 4,951,506 Toffolis) | **1.16x** | Official verifier, 102,400 shots, 0 errors |
| **dfthcsos** (chemistry) | Sample | Sample (not started) | 1.00x | n/a |
| **Combined (geometric mean)** | | | **~1.63x** | Estimate until the leaderboard scores it |

The dqi reference score is computed from the competition formula and the paper's numbers, not by running the sample file.

---

## Background

A fault-tolerant quantum computer pays a high price for every gate and every qubit. The most expensive gate is the **Toffoli**, so a circuit's cost is driven mostly by how many **qubits** it uses and how many **Toffolis** it runs.

The competition asks for three important sub-circuits to be rebuilt so they use fewer of both. These circuits use only reversible classical logic and phase gates, so a classical computer can check them. The official **verifier** runs each circuit on many inputs and confirms that:

1. the input is restored,
2. every scratch qubit returns to 0,
3. the phase is correct (within the allowed error).

The final score is the **geometric mean** of the improvements across the three tasks, so one weak task pulls the total down.

| Task | What the circuit does | Cost (lower is better) |
|---|---|---|
| invsqrt | Applies a phase of 1/sqrt(x) turns (Coulomb force between electrons) | qubits x (Toffolis + T gates / 4) |
| dqi | Decodes a Reed-Solomon code inside a quantum algorithm | repetition factor x qubits^2 x Toffolis |
| dfthcsos | Loads chemistry data for the FeMoCo molecule | lambda x Toffolis x qubits |

---

## Approach

### invsqrt: 2.08M to 0.555M (3.74x)

**Reference design.** Normalize x into a fixed range, guess 1/sqrt(x) with a cubic polynomial, then refine with one Newton-Raphson step in reversible fixed-point arithmetic.

**Profiling.** Counting Toffolis per stage showed the Newton step (four multiplications) took 62% of the circuit and the cubic polynomial 27%.

**Step 1: register-width tuning (about 5%).** A classical model of the circuit, validated against the simulator, made it possible to test every width setting in seconds. Four widths could each drop by one bit, giving 256 qubits and 7,447 Toffolis.

**Step 2: replace the Newton step with a lookup table (3.7x).** A table lookup costs about one Toffoli per entry, far less than a multiplication. With a large enough table the first guess is already accurate to 2^-24, so the Newton step is unnecessary. A search over table sizes and polynomial degrees found the best design: **384 rows (top 9 bits of the normalized input) with a quadratic polynomial per row**.

| Stage | Width-tuned reference | This work |
|---|---|---|
| Newton step | 4,606 | 0 |
| Polynomial | 2,028 | 1,778 |
| Table lookup | 0 | 551 |
| Normalize + denormalize | 810 | 810 |
| **Total Toffolis** | **7,447** | **3,139** |
| **Qubits** | **256** | **164** |

Removing the Newton step also frees the registers it needed, which is why qubits drop from 256 to 164.

**Verification.** The official verifier passes all 200,540,928 shots (every input from 1 to 3,133,452, 64 times each). The worst-case error uses 47% of the allowed tolerance.

### dqi: 5.73M to 4.95M Toffolis (1.16x)

**Reference design.** Reed-Solomon decoding with the Extended Euclidean Algorithm (222 steps), a Chien search over all 4,095 positions, then the first part run in reverse to clean up. The reference circuit was rebuilt exactly (1,893 qubits, 5,725,461 Toffolis) and each part was measured by switching it off.

| Part | Toffolis | Share |
|---|---|---|
| Field multiplications in the EEA | 2,430,600 | 42.5% |
| Register swaps | 1,107,264 | 19.3% |
| Inversions in the Chien search (4,095 x 315) | 1,289,925 | 22.5% |
| Rest of the Chien search | 154,812 | 2.7% |
| Divisions in the EEA | 167,832 | 2.9% |
| Counters, comparisons, addressing | 575,028 | 10.0% |

**Change.** Each position needs the inverse of a 12-bit field element, which the reference computes with five full multiplications (315 Toffolis). Building GF(2^12) as a tower over GF(2^6) turns one inversion into one 6-bit multiplication, one 6-bit inversion and two more 6-bit multiplications: **126 Toffolis**. Switching between the two representations costs no Toffolis. The idea is standard in classical field arithmetic; the work was making it exact and reversible in this setting.

**Result.** 4,951,506 Toffolis (exactly as predicted), the same 1,893 qubits, and a repetition factor of 1.1913, the same floor an exact decoder achieves. The official verifier reports 0 value errors and 0 phase errors over 102,400 shots.

---

## Correctness

1. **Classical model of each circuit** that predicts exact output bits, cross-checked against the kickmix simulator.
2. **Tests that must fail.** The harness reports failures on a deliberately broken decoder (130 of 256 shots), so a pass is meaningful.
3. **Exhaustive small tests.** The new inversion was checked on all 4,096 inputs, including zero: correct output, input unchanged, scratch qubits clean, no stray phase.
4. **Official verifiers** on the final files, plus a SHA-256 byte-for-byte check that the submission zip contains exactly the files that were tested.

**A bug caught along the way:** the first dqi build did not free the inversion's scratch qubits, leaking 12 qubits at each of the 4,095 positions (51,033 in total). The qubit count in the build output exposed it immediately.

---

## What did not work

- Lowering any invsqrt register width by two bits, or the coefficient width by one, breaks the 2^-24 error limit.
- A degree-1 table with 48 rows reaches only 2^-24.4 after a Newton step, too close to the limit once rounding is included.
- Letting the dqi decoder fail on some inputs is costly: at a 1% failure rate the repetition factor rises to 1.94, so the decoder was kept exact.
- dqi qubit count is unchanged: 840 hold the input and about 876 hold the second polynomial array, so this change only reduces Toffolis.

---

## Roadmap

- [ ] Submit this version and confirm the leaderboard score matches (it uses a secret salt for random test shots)
- [ ] **invsqrt:** binary barrel shifter for denormalization (est. ~300 fewer Toffolis), spend the spare error margin, tune coefficient rounding
- [ ] **dqi:** tower inversion for the EEA divisions (~1.7%), cheaper multiplier without extra qubits (63 to 54 Toffolis, ~6%), restructure the multiplication sweep (73 controlled multiplications per iteration, even when the control is off), which is the biggest remaining gain
- [ ] **dfthcsos:** not started; needs the starter kit and classical optimization of the Hamiltonian factorization

---

## Repository contents

| File | Description |
|---|---|
| `google-quantum-ai-verifiable-quantum-circuit-com.ipynb` | Full Kaggle notebook: profiling, classical models, circuit builders, official verifier runs, plots, submission packaging |

## Reproduce

1. Open the notebook on Kaggle with the competition data attached (`google-verifiable-quantum-circuit-compilation`).
2. Run the cells in order. The notebook builds the circuits, runs the official verifiers (`verify_submission_invsqrt.py`, `verify_submission_dqi.py`) and packages `submission.zip`.
3. Full verification runs take about 12 to 15 minutes for invsqrt and about 3.5 minutes for dqi.

## Tech stack

Python, NumPy, Matplotlib, **kickmix** (Qualtran circuit simulator), Kaggle notebooks.



## Author

**Gowtham Sevathan**
GitHub: [@sevathangowtham](https://github.com/sevathangowtham)
