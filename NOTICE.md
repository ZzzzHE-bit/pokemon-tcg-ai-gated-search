# Attribution

The heuristic portion of `main.py` is derived from Kiyota / The Pokémon
Company's Apache-2.0 official Mega Lucario sample agent. Local changes add
opponent-gated immunity routing, Stage-2 stadium tactics, deterministic hidden
state reconstruction, and a conservative one-turn Search API override.

Source: https://www.kaggle.com/code/kiyotah/a-sample-rule-based-agent-mega-lucario-ex-deck

The bundled `cg` Python interface and native libraries come from the official
competition runtime (`kaggle-environments==1.32.7`) and the official `cg-lib`
dataset. They are included to reproduce the competition agent package and
remain subject to their original license and competition terms.
