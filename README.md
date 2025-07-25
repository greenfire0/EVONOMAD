# EA‑NOMAD

**EA‑NOMAD** is a flexible evolutionary optimizer that couples global search (crossover + mutation) with local, derivative‑free refinement via the [NOMAD](https://www.gerad.ca/nomad) solver, exposed through **PyNomad**.

> **Why EA‑NOMAD?**
>
> * **Derivative‑free continuous control** – no action discretisation, no back‑prop through time; handles dense recurrence gracefully.
> * **Prior‑aware search** – starts from a biological connectome (or any strong prior).
>
>   * **Pure** mode: many tiny edits → minimal drift.
>   * **Hybrid** mode: < 50 edits → structural fidelity.
> * **Embarrassingly parallel** – flip on Ray to scale linearly with **CPU** cores.

---

## Installation

```bash
pip install EANOMAD

# Development version (clone + editable install)
git clone https://github.com/greenfire0/EA-NOMAD.git
cd EA-NOMAD
pip install -e .[ray]  # optional: Ray for parallel NOMAD calls
```

> **Requirements**: Python >= 3.9 · NumPy >= 1.23 · tqdm · PyNomad >= 0.9 · (optional: ray)

---

## Quick start

```python
import numpy as np
from EANOMAD import EANOMAD

obj = lambda x: -np.sum(x**2)  # maximise => global optimum at x = 0

opt = EANOMAD(
    "pure",                          # or "hybrid"
    population_size=32,
    dimension=10,
    objective_fn=obj,
    subset_size=5,
    bounds=0.2,
    max_bb_eval=100,
    n_mutate_coords=2,
)

best_x, best_fit = opt.run(generations=100)
print(f"Best fitness: {best_fit:.4f}")
```

---

## API

```python
EANOMAD(
    optimizer_type: Literal["pure", "hybrid"],
    population_size: int,
    dimension: int,
    objective_fn: Callable[[np.ndarray], float],
    subset_size: int = 20,
    bounds: float = 0.1,
    max_bb_eval: int = 200,
    n_elites: int | None = None,
    n_mutate_coords: int = 5,
    crossover_rate: float = 0.5,
    crossover_type: Literal["uniform", "fitness"] = "uniform",
    crossover_exponent: float = 1.0,
    init_pop: np.ndarray | None = None,
    init_vec: np.ndarray | None = None,
    low: float = -1.0,
    high: float = 1.0,
    use_ray: bool | None = None,
    seed: int | None = None,
)
```

---

## Methods

EA‑NOMAD offers two training strategies that differ only in *when* and *how* NOMAD is invoked within the evolutionary loop.

### Pure mode

Every generation, each individual in the population is passed to NOMAD for local refinement:

1. **Slice selection** – Pick `subset_size` coordinates at random (≤ 49, per NOMAD’s convergence guarantees).
2. **Local search** – Run PyNomad with a ±`bounds` hyper‑rectangle around that slice and a budget of `max_bb_eval` evaluations.
3. **Replacement** – If the refined individual improves its fitness, it replaces the original.
4. **Reproduction** – Select the top `n_elites` by fitness, then fill the rest of the population via fitness‑proportional crossover (probability `crossover_rate`) followed by random‑reset mutation (`n_mutate_coords` coordinates).

Pure mode tends to make *many* small synaptic adjustments, keeping the overall L2 distance to the original connectome low while steadily improving reward.

### Hybrid mode

An evolutionary mutation proposes a *sparse* change‑set first; NOMAD then fine‑tunes only those altered weights:

1. **Mutation** – Each offspring mutates a random subset of weights (usually < 50).
2. **Targeted NOMAD** – If the diff mask is novel and < 50 coords, run PyNomad *only* on that mask.
3. **Evaluation & elitism** – Update fitness, retain best individuals, proceed with crossover/mutation.

Hybrid mode yields comparable rewards to Pure mode while changing *far fewer* synapses – ideal when biological plausibility demands minimal rewiring.

Key hyper‑parameters (shared):

| name              | effect                                     |
| ----------------- | ------------------------------------------ |
| `subset_size`     | # parameters NOMAD refines per call (≤ 49) |
| `bounds`          | half‑width of the NOMAD search box         |
| `max_bb_eval`     | NOMAD evaluations per call                 |
| `n_mutate_coords` | coordinates reset per mutation             |

---

## Testing

```bash
pip install -e .[dev]  # includes pytest, ruff, black, etc.
pytest -q              # run smoke + reproducibility tests
```

---

## Contributing

1. Fork + create a feature branch
2. Run `pre-commit install`
3. Add unit tests for new behavior
4. PR + short summary of the change

---

## License

MIT License — see `LICENSE` file

---

### Acknowledgements

* Hi my name is miles, I hope you enjoy these algorithms and optimize some cool shit using them <3
* [PyNomad](https://github.com/bertsky/pynomad)
* NOMAD team at Polytechnique Montréal / GERADmiddlemouse
