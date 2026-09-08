# When Rules Break Heuristics: Gated Search for Mega Lucario ex

**Subtitle:** An effect-aware rule agent that spends exact simulation only when visible card text changes the winning line

## 1. Motivation

Pokémon TCG is a partially observed, long-horizon game, but many decisions are
easy once the engine has produced the legal action set. The difficult cases are
where a locally reasonable score is invalidated by card text: a Pokémon ex
attacks into immunity, a 320-HP Stage 2 narrowly survives 300 damage, or an
apparently useful action order destroys the only winning attack line.

Our thesis is simple: retain a stable rule policy for ordinary play, encode
small causal exceptions when the relevant card is visible, and invoke the
official simulator only inside those exceptional matchups. This gives us an
interpretable hybrid with a legal fallback on every decision.

## 2. Deck choice

We use the official 60-card Mega Lucario ex sample shell: 16 Pokémon, 31
Trainers, and 13 Fighting Energy. Mega Lucario ex supplies efficient 130/270
damage and acceleration; Hariyama and Solrock are one-Prize alternate attackers.
Premium Power Pro and Gravity Mountain create exact knockout thresholds.

| Deck section | Cards |
|---|---|
| Pokémon (16) | 2 Makuhita, 2 Hariyama, 2 Lunatone, 3 Solrock, 3 Riolu, 4 Mega Lucario ex |
| Trainers (31) | 4 Dusk Ball, 2 Switch, 4 Premium Power Pro, 4 Fighting Gong, 4 Poke Pad, 1 Hero's Cape, 2 Boss's Orders, 4 Carmine, 4 Lillie's Determination, 2 Gravity Mountain |
| Energy (13) | 13 Basic Fighting Energy |

We screened five conservative ratios over four opponents (250 games per cell):
the original list, a third Gravity Mountain replacing Poke Pad or Energy, a
3-3 Hariyama line, and a combined variant. The small screen produced unstable
rankings and no variant improved every axis, so we kept the official list. This
was a deliberate rejection: a policy improvement should not be confused with
a favorable deck sample.

## 3. Policy

The agent has three layers.

**Stable heuristic.** We begin with Kiyota and The Pokémon Company's official
Apache-2.0 Mega Lucario sample. It ranks only engine-provided legal options and
plans an attacker, target, energy attachment, switch, evolution, and attack.
Our entry point wraps every added component in exception handling and returns
the heuristic action if search cannot run.

**Causal card-text gates.** When a visible defender prevents damage from
Pokémon ex (including Crustle), the policy assigns zero projected Lucario
damage, shifts energy toward Makuhita/Hariyama, promotes the evolution, and
routes the attack through a one-Prize Pokémon. The gate is inactive in ordinary
matchups.

Against a visible Stage 2, the agent values Gravity Mountain before damage
planning. This matters against Dragapult ex: Gravity Mountain changes 320 HP to
290 HP, while Mega Brave (270) plus Premium Power Pro (30) reaches 300. The
three-card interaction converts a near miss into a three-Prize knockout.

**Gated one-turn search.** Search activates only after a Dragapult line or an
ex-immunity defender becomes public. We subtract our visible hand, discard,
board, evolutions, tools, Energy, and Stadium from the 60-card list, then use a
deterministic shuffle to form one hidden-state realization. We make no claim to
know the opponent's unrevealed cards; conservative filler cards preserve zone
sizes because the rollout ends before the opponent acts.

For each legal main-phase action, the official Search API clones the current
state. The stable heuristic completes the rest of our turn. A leaf score uses
terminal result, Prize differential, Prize-weighted board presence, remaining
HP, attached Energy, hand size, and deck size. Search may replace the heuristic
choice only when its value is at least 250 points higher. In the 1,000 gated
games of the final run, search made 21,479 calls, changed 3,271 decisions (15.2%), and
failed zero times. This is deterministic one-turn lookahead, not full MCTS.

```text
a0 = heuristic(state)
if no visible exception: return a0
for each legal main action a:
    clone = SearchAPI.step(root, a)
    leaf  = heuristic_rollout_to_turn_end(clone)
    value[a] = prize_and_board_value(leaf)
return argmax(value) only if value[best] >= value[a0] + 250 else a0
```

## 4. Evaluation protocol

All tests use the official native battle core from `kaggle-environments
1.32.7`. Every game starts with fresh policy memory; seats alternate; draws are
excluded only from the win-rate denominator; exceptions and illegal actions
are recorded as errors. The native library exposes no seed hook, so independent
runs are not paired. We report Wilson 95% intervals and use complete official
rule agents as opponents wherever available.

| Frozen matchup | Games | Win rate | 95% CI | Errors |
|---|---:|---:|---:|---:|
| Search Lucario vs official Dragapult | 500 | 63.1% | 58.7–67.2% | 0 |
| Search Lucario vs official Lucario | 500 | 50.0% | 45.6–54.4% | 0 |
| Search Lucario vs official Abomasnow | 500 | 52.6% | 48.2–56.9% | 0 |
| Search Lucario vs Crustle stress deck | 500 | 91.4% | 88.6–93.6% | 0 |

The ablation tells the clearer story. Official Lucario won 53.4% against the
official Dragapult agent (500 games). Adding causal gates and Stage-2 threshold
logic reached 56.6%; gated search reached 63.1%, CI 58.7–67.2 (500 games each).
Against the Crustle stress deck, the same progression was 36.0% (1,000 games),
84.0%, and 91.4%, CI 88.6–93.6 (500 games each). All reported runs had zero agent
errors.

## 5. A useful failure

Our first independent deck was Mega Starmie ex. It appeared strong against
native option-order opponents: 55.2% against Abomasnow and 82.8% against a
Crustle stress deck. That conclusion collapsed when we imported complete
official policies. Starmie scored only 34.0% against Dragapult, 25.6% against
Lucario, and 20.2% against Abomasnow (500 games each).

This reversal changed the entire workflow. Weak legal opponents are useful for
crash testing, but they are not evidence of strategic strength. We therefore
kept Starmie as a negative result, upgraded the opponent pool, and froze a
candidate only after strong-policy validation.

## 6. Scope and next steps

The largest limitation is external validity. We did not enter the earlier
Simulation track, so this agent has no official final-ladder score; local wins
against public agents cannot substitute for the hidden population. The
Crustle deck is deliberately adversarial but uses a simple opponent policy.
Search uses one hidden-state realization and one-turn rollouts, so it does not
solve the full partially observed game.

The strongest published solutions point toward the next step. The 15th-place
team trained 7.5M-parameter recurrent actor-critic specialists through
population self-play, while the 27th-place report used PPO, curriculum, and an
entity transformer. We would retain our rule gates as a safety prior, replace
the hand-built leaf value with a recurrent population-trained critic, and
search over several belief-consistent determinizations.

## 7. Reproducibility and attribution

The supplied archive contains `main.py`, `deck.csv`, the official cross-platform
`cg` runtime, Apache 2.0 license text, and attribution. The build script removes
AppleDouble files and bytecode, checks the 60-card deck and required binaries,
and records SHA-256. A separate extraction test imported the packaged agent and
completed 20 native-engine games with zero errors.

Official API documentation: https://matsuoinstitute.github.io/cabt/  
Official Lucario sample: https://www.kaggle.com/code/kiyotah/a-sample-rule-based-agent-mega-lucario-ex-deck  
Official Dragapult sample: https://www.kaggle.com/code/kiyotah/a-sample-rule-based-agent-dragapult-ex-deck  
15th-place solution: https://www.kaggle.com/competitions/pokemon-tcg-ai-battle/writeups/15th-place-solution  
27th-place solution: https://www.kaggle.com/competitions/pokemon-tcg-ai-battle/discussion/738158
