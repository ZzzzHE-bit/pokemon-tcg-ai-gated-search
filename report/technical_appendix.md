# Technical appendix

## Agent boundary

The competition entry point is the last `agent(obs_dict) -> list[int]`
function in `submission/main.py`. It has no third-party Python dependency. The
only runtime dependency is the supplied `cg` package and native library.

On the initial call, the agent returns the 60 integers from `deck.csv`. On each
game decision it first obtains the official heuristic action. Search is allowed
to change that action only in main phase, from turn two onward, with one choice
required, 2–18 legal options, and a visible card ID from the exception set.
Every conversion, inference, or native search failure returns the legal
heuristic action.

## Leaf value

For owner `i`, the one-turn leaf score is:

```text
1,000,000 * terminal_result
+ 6,000 * (opponent_prizes - own_prizes)
+ own_board_value - opponent_board_value
+ 25 * (own_hand - opponent_hand)
+ 3 * (own_deck - opponent_deck)
```

Each Pokémon contributes `650 * prize_value + HP + 90 * attached_energy + 35
* tools`. The heuristic wins ties. An override needs a 250-point margin.

## Evaluation stages

| Stage | Purpose | Result source |
|---|---|---|
| Native legal agents | Crash and action-shape screening | `results/final/` |
| Full public policies | Reject false strategic gains | `results/strong_pool_v1/` |
| Starmie native screen | Reproduce the rejected weak-opponent signal | `results/starmie_native/` |
| Rule-gate ablation | Isolate immunity and HP-threshold rules | `results/hybrid_v1/`, `results/hybrid_v2_gravity/` |
| Search development | Decide whether added runtime is justified | `results/search_probe_v1/` |
| Search confirmation | Frozen 400+400 confirmation | `results/search_validation_v2/` |
| Final gated matchups | Dragapult and Crustle, 500 games each | `results/final_strong/` |
| Final entry-point check | Lucario and Abomasnow, 500 games each | `results/final_actual_agent/` |
| Final causal ablation | Search-disabled policy, 500 games each | `results/final_ablation/` |

## Commands

```bash
PYTHONPATH=vendor:. .venv/bin/python -m experiments.strong_pool \
  --games 500 \
  --only search_vs_official_dragapult search_vs_native_crustle_stress \
         search_vs_official_lucario search_vs_official_abomasnow \
  --output-dir results/final_actual_agent

.venv/bin/python src/build_submission.py
.venv/bin/python src/verify_submission.py --games 20
MPLCONFIGDIR=/private/tmp/ptcg-mpl .venv/bin/python src/make_figures.py
```

## Runtime and reproducibility limits

The native engine does not export a random seed setter on this macOS build.
The Python `--seed` option affects only the deliberately random baseline.
Consequently, repeated policy comparisons are independent samples rather than
paired games. Alternating seats limits first-player imbalance, and Wilson
intervals communicate binomial uncertainty.

Search averages roughly 0.8 seconds per complete Dragapult game in local
validation, versus roughly 0.05 seconds for heuristic-only games. This is well
below the observed local game budget, but no official Simulation submission
was available to verify hidden production hardware or ladder behavior.
