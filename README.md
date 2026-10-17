# Quantum Lattice Boltzmann Method for 2D Fluid Flow

This repository contains a **PennyLane-based quantum-circuit prototype of the Lattice Boltzmann Method (LBM)** for a two-dimensional flow field with a square obstacle. The notebook implements quantum-circuit operations corresponding to collision, bounce-back, and streaming, and compares the reconstructed distribution functions with a classical LBM implementation on a small lattice.

> **Scope note:** The quantum/classical comparison currently runs on a $4\times4$ lattice for two time steps using a simulated quantum circuit and explicit unitary-matrix extraction. The notebook also includes a longer, $64\times64$, 8,000-step **classical-only** simulation. The project is a prototype for exploring a quantum-circuit formulation of LBM; it does not demonstrate quantum hardware execution or a quantum computational advantage.

---

## Overview

The Lattice Boltzmann Method models fluid evolution using particle distribution functions defined on a discrete lattice. In this project, a two-dimensional, nine-velocity lattice (D2Q9) is used to study flow around a centrally located square obstacle.

The notebook explores how the main distribution-function updates can be represented with quantum operations:

- **Collision:** a PREPARE–SELECT–PREPARE† / linear-combination-of-unitaries-inspired construction combines the current distribution and the equilibrium distribution.
- **Bounce-back:** the distribution associated with the opposite lattice direction is used at obstacle cells.
- **Streaming:** controlled permutation operators shift distribution values between neighboring lattice sites with periodic indexing.
- **Classical macroscopic updates:** density, velocity, and the equilibrium distribution are calculated classically and supplied to the next circuit iteration.

The circuit uses controlled $R_Y$ rotations to encode individual distribution values as amplitudes on a value qubit. Lattice direction and position are represented by address registers.

## Main Features

- D2Q9 Lattice Boltzmann model.
- Quantum-circuit prototype implemented with **PennyLane**.
- Controlled-rotation encoding of distribution-function values.
- LCU-inspired collision construction with normalization for the collision coefficients.
- Bounce-back treatment of a central obstacle.
- Controlled row and column permutations for streaming.
- Classical LBM baseline for numerical comparison.
- Visualization of the vorticity field.
- A longer classical simulation for observing the flow over more time steps.

---

## Physical Model and Parameters

The implementation uses the following settings in the notebook:

| Parameter | Value / definition |
|---|---:|
| Lattice model | D2Q9 |
| Initial horizontal velocity, $U_0$ | 0.15 |
| Relaxation time, $\tau$ | 0.51 |
| Collision parameter, $\Omega$ | $1/\tau$ |
| Initial density, $\rho$ | 1 |
| Initial vertical velocity, $u_y$ | 0 |
| Obstacle | Central $2\times2$ block of lattice cells |
| Streaming boundary treatment | Periodic indexing |

The initial horizontal velocity includes a small sinusoidal perturbation to introduce spatial variation. The horizontal and vertical velocities are set to zero in the obstacle mask during initialization.

The equilibrium distribution is computed classically using the standard second-order D2Q9 equilibrium expression implemented in the notebook:

$f_i^{\mathrm{eq}}=w_i\rho\left[1+3(\mathbf{c}_i\cdot\mathbf{u})+4.5(\mathbf{c}_i\cdot\mathbf{u})^2-1.5|\mathbf{u}|^2\right].$

Here, $w_i$ denotes the lattice weight, $\mathbf{c}_i$ is the discrete velocity for direction $i$, and $\mathbf{u}=(u_x,u_y)$ is the macroscopic velocity.

---

## Quantum Circuit Design

### Qubit Registers

For the quantum demonstration, the lattice size is $n=4$. The circuit uses **10 qubits** arranged as follows:

| Register | Number of qubits | Role |
|---|---:|---|
| `IDX` | 4 | Addresses one of the nine D2Q9 directions |
| `WHICH` | 1 | Selects the current distribution or equilibrium distribution during collision |
| `ROW` | 2 | Encodes the lattice row |
| `COL` | 2 | Encodes the lattice column |
| `VAL` | 1 | Encodes a distribution value through the amplitude of its $|1\rangle$ state |

The `IDX`, `ROW`, and `COL` registers provide the address of a distribution element. The `WHICH` register distinguishes between the current distribution $f$ and the equilibrium distribution $f^{\mathrm{eq}}$. The `VAL` qubit receives controlled $R_Y$ rotations.

### Distribution-Value Encoding

For a distribution value $v$, the code uses angle encoding of the form

$\theta=2\arcsin\left(\frac{v}{\mathrm{scale}}\right),$

where the value is clipped to the valid range before calculating the angle. Consequently, the amplitude of the $|1\rangle$ component is proportional to the scaled value.

This is an **addressed, element-by-element encoding**. It is not amplitude encoding of the entire distribution tensor in a single quantum state.

### Collision

The classical BGK collision update has the form

$f_i^\*=(1-\Omega)f_i+\Omega f_i^{\mathrm{eq}}.$

The quantum circuit uses the normalization

$\Lambda=|1-\Omega|+|\Omega|$

to construct the normalized linear combination in its `WHICH` branches. The signs of the collision coefficients are incorporated in the controlled rotations. During simulator-side extraction, the known normalization is removed to reconstruct the physical post-collision distribution.

### Bounce-Back

At cells marked as part of the obstacle, the circuit directly encodes the distribution from the opposite lattice direction, using the `OPPOSITE` direction mapping. This branch does not use the collision LCU normalization.

### Streaming

For each lattice direction, the row and column registers are shifted according to the corresponding discrete velocity. The shifts are represented with controlled permutation unitaries and use periodic indexing:

$(r,c)\mapsto((r+c_x)\bmod n,(c+c_y)\bmod n).$

### Classical Macroscopic Quantities

The equilibrium distribution and the macroscopic density and velocity are calculated classically. The updated values are then used to define the input data for the next circuit iteration. The notebook is therefore a **hybrid classical–quantum simulation workflow**, even though the distribution-function transformations are represented by a quantum circuit.

---

## Classical Baseline and Numerical Comparison

The notebook contains two classical LBM experiments and one small-grid quantum/classical comparison.

### Small-Grid Comparison

The quantum and classical implementations are both run on a $4\times4$ lattice for **two time steps**. The notebook compares the resulting distribution tensors using the mean absolute error:

$\mathrm{MAE}=\operatorname{mean}\left(|f_{\mathrm{classical}}-f_{\mathrm{quantum}}|\right).$

For the saved notebook run, the reported value is:

| Metric | Result |
|---|---:|
| Mean absolute error in the distribution tensor | $1.45\times10^{-17}$ |

This is near floating-point precision for the tested configuration. It is a numerical consistency check for this specific small-grid run, not evidence of scalability, a hardware result, or quantum advantage.

### Larger Classical Run

A separate section runs the **classical implementation only** with these settings:

| Parameter | Value |
|---|---:|
| Lattice size | $64\times64$ |
| Number of time steps | 8,000 |
| Plot interval | 1,000 steps |

This longer run illustrates the classical LBM evolution and produces vorticity visualizations. The quantum-circuit section is not run on this $64\times64$ grid.

---

## Visualization

The notebook computes the vorticity from spatial derivatives of the macroscopic velocity components:

$\omega_z=\frac{\partial u_y}{\partial x}-\frac{\partial u_x}{\partial y}.$

It plots the vorticity field and overlays the obstacle mask, allowing qualitative inspection of the flow field as the simulation progresses.

---

## Software and Libraries

The notebook uses:

- **Python** — implementation language.
- **NumPy** — arrays and numerical operations.
- **PennyLane** — quantum-circuit construction and unitary-matrix simulation.
- **Matplotlib** — plotting and visualization.
- **IPython** — notebook display utilities.
- **Jupyter Notebook or JupyterLab** — interactive execution environment.

The notebook does not pin dependency versions. For a reproducible repository, record the versions tested in your environment or add a `requirements.txt` file.

### Installation

Create and activate a Python environment, then install the dependencies:

```bash
python -m pip install numpy pennylane matplotlib ipython jupyter
```

### Running the Notebook

1. Clone or download the repository.
2. Install the dependencies listed above.
3. Open the notebook in Jupyter Notebook or JupyterLab.
4. Run the cells in order, from parameter setup through the quantum/classical comparison.
5. Run the final section to execute the longer classical simulation.

For a clean repository, consider renaming the uploaded notebook to `Quantum_LBM.ipynb` before committing it.

```bash
jupyter lab Quantum_LBM.ipynb
```

---

## Repository Structure

A simple starting structure is:

```text
quantum-lattice-boltzmann-method/
├── Quantum_LBM.ipynb
├── README.md
├── requirements.txt   # optional; recommended for reproducibility
└── LICENSE             # add after choosing a license
```

The current implementation and experiments are contained in the notebook. The `requirements.txt` and `LICENSE` entries above are suggested additions, not files supplied with the uploaded notebook.

---

## Computational Limitations and Future Work

The current notebook is a proof-of-concept with several important limitations:

- **Small quantum grid:** the quantum comparison is configured for a $4\times4$ lattice.
- **Explicit unitary construction:** the simulation obtains the complete circuit matrix using PennyLane's `qml.matrix` and reads selected matrix elements to reconstruct the distribution tensor. The resulting unitary is a dense $2^{10}\times2^{10}$ matrix for the current 10-qubit configuration.
- **No hardware execution:** the notebook does not run the circuit on a physical quantum processor. Its discussion of measurement-based extraction describes a possible hardware-oriented approach, but the implemented extraction uses the simulator's explicit unitary matrix.
- **Classical outer loop:** macroscopic density, velocity, and equilibrium distributions are calculated classically between circuit iterations.
- **No demonstrated quantum advantage:** the matching result is limited to the reported configuration and should not be interpreted as a speedup or reduction in end-to-end computational cost.

Useful next steps include separating the implementation into reusable Python functions, adding automated tests across several small-grid configurations, pinning package versions, measuring runtime and memory use, and implementing and validating a measurement-based workflow before making claims about execution on quantum hardware.

---

## License

This project is licensed under the **Apache License 2.0**.
You may use, modify, and distribute this software in accordance
with the terms of the license.

See the [LICENSE](LICENSE) file for the complete license text.

If you use this software in your research or publications,
please cite this repository using the information provided in
[CITATION.cff](CITATION.cff).
