# Back Track King — CodinGame Summer Challenge 2026

**English** | [Français](README.fr.md)

| | |
|---|---|
| **Contest** | [CodinGame Summer Challenge 2026](https://www.codingame.com/contests/summer-challenge-2026-back-track-king), September 7–21, 2026 |
| **Game** | Back Track King: two players build railways on a grid to connect towns, simultaneous turns |
| **Result** | Legend league, the top league |
| **Final bot** | An ensemble of two neural networks of my own design (S3z), trained by imitation, running in C++ |
| **Stack** | C++17 (bots, simulator), Python and PyTorch (simulator, training), Java (official referee) |

## The game

- Two players lay rails on a grid (from 21×14 to 30×20). Towns want to be connected to other towns.
- Each turn, a paint budget of 3 lets a player lay up to 3 rails (plain 1, river 2, mountain 3). The budget is refilled each turn, not saved: paint not spent is lost.
- Once per turn, a player can also disrupt a zone. Each disruption raises its instability. Past a threshold, the zone is destroyed: its rails are erased and nothing can be built there anymore.
- Income: for each pair of towns that want to be connected and are linked, the shortest path counts. Each player earns, every turn, the number of cells they own on these paths.
- Three consequences: a long path that I own pays more than a short one; rails can be used by both players, so completing a connection whose path is mostly the opponent's pays them; and connecting early is worth more than connecting a lot.
- Both players play at the same time, with 50 ms per turn.

## The final bot in brief

Each turn:

1. Two neural networks of my S3z architecture read the grid. They were trained separately, and they play differently.
2. Their scores for each cell, for laying a rail and for disrupting, are averaged.
3. The bot turns these scores into a full turn: up to 3 rails within the paint budget, and the zone to disrupt.

There is no search at play time: every attempt to rerank the network's moves lost (see below). The weights are quantized to 7 bits, so that the two networks fit in a single C++ file under the 100,000-character limit.

## Timeline

| Version | Date | Main idea | League |
|---|---|---|---|
| 1 | September 8–9 | Beam search and disruption, on an exact simulator | Wood → Bronze |
| 2 | September 10–16 | Beam v4: corridors, corridor bonus, anti-disruption tuning | Bronze → Silver |
| 3 | September 19–20 | S3z, my own imitation network | Gold |
| 4 | September 20 | Ensemble of two S3z networks | Gold → Legend (final version) |

## Bot versions

### Version 1 — Beam search and disruption

**The base: an exact simulator.** A C++ model of a full turn (income, rails, collisions, disruptions, destructions, connections), checked against the official referee: 1,500 random games, 150,000 turns compared, 0 differences.

**The beam search.**
- The bot builds its turn rail by rail, and keeps the best partial plans at each step.
- The candidates are the cells on the scored paths of the most valuable connections. Bounding them (8 connections, 64 cells) brought the worst decision time from 718 ms down to 42 ms.
- The bot builds exactly the path the referee scores: the shortest path with the referee's tie-break order (north, east, south, west). All my rails then lie on scored paths. Against my greedy bot (which also disrupts), the win rate went from 64.5% to 74.5%.

**Disruption by denied income.** The bot disrupts the zone whose destruction would remove the most income from the opponent, counting only the connections that really score, and favoring zones close to destruction. Against the same greedy bot: 72.5% → 78.0% (200 games).

**The method.** One change at a time, each with a GO / NO-GO verdict: a change is kept only if the win rate goes up, the slowest decision stays under 47 ms, there is no crash or illegal action, and the simulator stays exact.

### Version 2 — Beam v4: corridors and anti-disruption tuning

- **Corridors:** the beam also considers routes close to the shortest one, taking into account who owns each cell.
- **Corridor bonus:** a cell used by many desired corridors gets a bonus, both when ranking the connections and when ranking the candidate cells.
- **Time guard:** every scan is bounded from the start of the turn, because the referee disqualifies a bot that takes more than 50 ms.
- **Anti-disruption tuning:** three constants changed, and no new code. The bot avoids building in zones whose instability is rising, and gives less weight to immediate gain. It beat the previous version in 61.3% of 390 games on held-out maps.

**The ceiling.** After that, 11 more ideas on top of this beam (routing, corridors, contested zones, disruption weights, leading or trailing...) were all rejected or tied. The search had reached its ceiling.

### Version 3 — S3z, my own neural network

**The design.**
- Decisive measurement: the network takes 97% of the time in C++. The input features are free, and the number of operations is the real constraint.
- The S3z architecture: dilated separable convolutions, plus a block that adds context at the level of zones ("ZoneContext"). 58,693 parameters.
- ZoneContext was predicted to be "almost free" at 0.05 ms, but measured at 0.59 ms per block: an error by a factor of 20, because it is limited by memory access, not by computation. It is still worth about 8 points of win rate, measured against the same network without it (on a single training seed).

**My own simulator, in Python.** It is written from the referee's specification and checked by two independent oracles: a differential test against the official Java referee, compiled without changes, and 14,000 real replays. It caught a real bug: Java's `String.split` drops trailing empty strings, Python's does not, and one bot ends its output with a `;`. No synthetic test would have found it.

**Training.** By imitation (behavior cloning) of the moves of top players, from replays downloaded from CodinGame. Two independent trainings are equally strong but play differently: they agree on only 22.9% of their commands. This fact is the basis of version 4.

**In C++.** 86,795 characters. Under 71,250 parameters, 16-bit floats are enough: no quantization needed. The C++ decisions are identical to Python on 100% of 2,000 states, 2,958 turns and 48 full games in 32-bit, and on 99.66% in 16-bit, the gap coming only from rounding.

### Version 4 — Ensemble of two S3z networks (final version)

- The two networks' outputs are averaged. Against a single S3z, the ensemble wins 54.6% of 2,000 games (95% confidence interval: 52.4–56.7%).
- Averaging two epochs of the same training gives nothing (44.75%): the gain comes from diversity.
- Quantized to 7 bits, the ensemble stays clearly better than a single S3z (McNemar test, p = 0.003), and quantization costs nothing measurable (p = 0.91 over 2,500 seeds).
- The first 7-bit ensemble fit in 87,415 characters, with decisions identical to Python on 2,958 out of 2,958 commands and 48 out of 48 games, and took 31 ms per turn out of 50.
- **Result:** promoted to Legend.

## What didn't work

On the search side (beam):
- **Predicting the opponent inside the beam (one round):** −12 to −14 points, and over the time budget. The bot became too timid and gave up connections it would have won.
- **Building another path of the same length that reuses my rails:** −4.5 points. The referee keeps scoring its own path, so my "optimized" rails fell off it.
- **An explicit bonus for "backbone" cells shared by several connections:** about 50%, no signal. The beam already rewards them.
- **More search time:** a 40 ms budget did worse. On average, the beam already explored all its candidates in 4.3 ms.

On the network side (a research session of 26 experiments):
- **Reranking the network's moves:**
  - by the cost of completing connections: wrong objective, since income comes from owning cells, not from connecting;
  - by the real income: this short-sighted optimization loses 25 points;
  - by the network's value output: values vary 7 times less between the moves of one turn than between turns, so the value cannot tell those moves apart.
- **Not spending the paint when the network is unsure:** unspent paint is lost.
- **A grid + graph network (S4):** 46.8% against S3z. The graph block brings nothing.
- **Weight averaging (SWA):** wrong protocol (learning rate ending at a tenth of its value).
- **Distilling the ensemble into a single network:** with hard targets, it memorizes a narrow target and generalizes badly; with soft targets, the error over 600 outputs is dominated by noise.
- **In total, eight attempts to train a single network better than S3z all failed.**

## Where the limit is

- **For the search,** the limit was not speed: more time brought nothing, and all tuning was exhausted. The only room left was anticipating the opponent, for a large effort and an uncertain gain.
- **For the network,** I rebuilt exactly 77 real CodinGame games of S3z (77 official scores out of 77 reproduced):
  - it won 40% of its games against the top 30 of Gold, 70% against the middle of Gold, and 88% against the bottom;
  - in my losses, the opponent does not score more: I score half as much;
  - the clearest sign is the income difference at turn 15: −5.6 in losses, +4.6 in wins (p = 0.001).
- **The next step** would be to rank the moves of a single turn against each other. That needs examples of which move is better, so simulated continuations of the game (rollouts).

## Tooling

- **An exact C++ simulator of a turn,** checked against the official referee: 150,000 turns, 0 differences.
- **A Python simulator,** checked by two independent oracles: the official referee and 14,000 real replays.
- **A local arena** with shared seeds, both sides played, and a strict GO / NO-GO protocol.
- **A partitioned seed plan** (screening, second level, confirmation, outside opponents): no seed used to choose was used to conclude.
- **Python ↔ C++ parity tests,** decision by decision.
- **Exact reconstruction of real CodinGame games:** the map cannot be regenerated from the seed, so it is read from the replay.
- **Volume:** about 45,000 games in the last research session.

## What I learned

- **Understand the scoring rule before optimizing anything.** A long path that I own pays more, and paint not spent is lost: these two facts conditioned everything.
- **Mirror the referee exactly.** Its tie-break order is the target, not a detail.
- **Never conclude under 2,000 seeds, and always compare on the same seeds.** A 100-seed filter was wrong seven times.
- **Measure instead of predicting.** ZoneContext cost 20 times more than predicted, and the network, not the features, took 97% of the time.
- **What must be identical between Python and C++ is the decision, not the bits:** the bits change with the number of threads.
- **Two equally strong networks that play differently are worth more together** than any single network I could train.
