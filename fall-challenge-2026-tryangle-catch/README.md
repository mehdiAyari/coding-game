# TryAngle Catch — CodinGame Fall Challenge 2026

**English** | [Français](README.fr.md)

| | |
|---|---|
| **Contest** | [CodinGame Fall Challenge 2026](https://www.codingame.com/contests/fall-challenge-2026), November 2–9, 2026 |
| **Game** | TryAngle Catch: two players, simultaneous turns, on a graph of houses |
| **Result** | Legend league, the top league |
| **Final bot** | A graph neural network trained by imitation of top players, plus a one-turn search with an exact simulator |
| **Stack** | C++17 (bot, simulator, analysis tools), Python and PyTorch (training), Java (official referee) |

## The game

- The map is a graph of houses linked by paths. Three houses that are all linked together form a triangle.
- Each player starts with 3 units. Each turn, units can move to a neighboring house.
- A player owns a house when they have more units on it than the opponent. Owning the three corners of a triangle captures it.
- A captured triangle can be used once: to spawn a unit, to attack a neighboring house (all enemy units on it die), or to add or remove a path. It then becomes neutral again.
- A unit surrounded by enemy houses dies.
- Each triangle owned gives 1 point per turn. Owning 80% of the triangles wins at once.
- Both players play at the same time, with 50 ms per turn.

## The final bot in brief

Each turn:

1. A graph neural network reads the board and proposes complete plans: an order for every stack of units and an action for every usable triangle.
2. It proposes up to 24 plans for me and up to 8 for the opponent, seen from their side of the board. Hand-written variants are added: my triangles attacking, and a "maximum threat" reply from the opponent.
3. An exact simulator plays one turn for each pair of plans (mine against theirs).
4. The network's value output scores each resulting position.
5. A plan's score mixes its worst case (30%) and its average (70%) over the opponent's plans. Weak plans are dropped along the way, to spend more time on the good ones.
6. The bot plays the best plan. It stops after 36 ms, out of the 50 ms allowed.

The whole bot is a single C++ file of at most 100,000 characters, network included. The weights are quantized to 8 bits and packed 15 bits per Unicode character. The network runs in hand-written C++ with AVX2 instructions.

## Timeline

| Version | Date | Main idea | League |
|---|---|---|---|
| 1 | November 3 | Hand-tuned classic search | Bronze → Silver |
| 2 | November 4–5 | Search guided by a small imitation network | Silver → Gold |
| 3 | November 7 | Imitation network + one-turn search | Gold → Legend |
| 4 | November 7 | More data, better candidate plans, attacks | Legend (final version) |

## Bot versions

### Version 1 — Hand-tuned classic search

**Principle.** Each turn, the bot builds a full plan: an order for every stack of units (move to a neighbor or stay) and an action for every usable triangle (spawn or attack).

**How it chooses.**
- It starts from a greedy plan: teams of 3 units toward the cheapest triangles to take.
- It changes the plan step by step and keeps the changes that improve the score (hill climbing).
- It alternates with the opponent within the turn. It looks for the opponent's best reply to my plan and remembers it, up to 4 replies per turn. Then it improves my plan against these replies, focusing on the worst one.
- Every plan is simulated exactly against these replies.
- The score comes from a hand-written evaluation: triangles owned, units, threats, territory, and a "capture potential" (how many turns are needed to take each triangle).

**Tuning by hand.** I studied my losses, changed a weight or added a term, then tested on hundreds of local games.

**The ceiling.** After a few versions, no change passed anymore: more stacks, an attack bonus, a penalty for units that block a triangle... all rejected.

### Version 2 — Search guided by imitation

**The idea.** When hand tuning hit its ceiling, I added a "guide": a small network that imitates the moves of strong players.

**The guide.**
- A small graph neural network: 3 layers, 32 values per house, about 30 input features per house.
- Trained on games of strong players. For each stack, it gives the probability of each destination.
- It predicted their moves better than my own search did: 51% of units sent to the right house, against 40%.
- It costs almost nothing: computed once per turn for each side, in 0.28 ms.

**How the search uses it.**
- **For my plans:** a plan gets a small bonus when its moves are likely for the guide. The search stays free, but it tries first the moves a good player would make.
- **For the opponent:** the same guide, applied from their side of the board, steers the search for their replies. Their moves are predicted better.
- **The bonus must stay small.** Too strong, the bot copies the guide and gets much weaker.

**Result.** Against the same search without the guide, it won about 2 games out of 3. This version took the bot to Gold.

**Making the search faster.**
- **Measure first:** a profile with the CPU cycle counter showed 74% of the time in the evaluation, and about 60% in the capture potential of both players.
- **A fast and exact simulation:** the most common case (moves to a neighbor and spawns) takes a short path, checked identical to the full engine by a tool. A simulated turn went from about 2.5 µs to under 1 µs.
- **A faster capture potential, still exact:** the houses at each distance from each corner are stored in advance as bitmasks, and one shared board with an undo journal replaces a board copy for every triangle.
- **A cache of scores:** within a turn, the same "my plan, their plan" pair comes back often. 63% of scores come from the cache.
- **In total,** per turn, in 38 ms: about 7,200 evaluations computed, plus 12,300 scores taken from the cache.
- **Rejected:** an approximate potential gave 45% more evaluations, but the bot took far fewer triangles. My rule: speed up only if the decisions don't change.

At this point, speed was no longer the limit: with 10% less time, the bot played just as well (598 wins to 588 over 1,200 games). The limit was the evaluation.

**The new ceiling.** I tried 4 other guides, including 3 trained on Gold games. They predicted moves better, but none won more games (600 games each). Hand tuning hit its ceiling again, so I moved to a much bigger network that proposes full plans and judges positions.

### Version 3 — Imitation network + one-turn search

**The data.**
- Thousands of games of top players, downloaded through CodinGame.
- My simulator replays each game move by move. That gives the positions and the moves really played by each player: 1 to 2.5 million positions.
- I kept mostly the moves of the top 30 players of the leaderboard.

**What the network sees.**
- The board as a graph: houses linked to their neighbors, and triangles made of three houses.
- About 68 features per house: units of both players, owner, neighbors, threats, distances.
- About 33 features per triangle: units on its corners, owner, capture possible next turn, risk of attack.

**The network.**
- A graph neural network with 4 layers and 64 values per house. At each layer, a house receives information from its neighbors and from the triangles it belongs to.
- Three outputs: for each stack, a probability for each destination; for each triangle, a probability for each action; and a value, the estimated chance of winning.
- **Coordinated decoding:** a plan is not decided at once, but in 6 rounds. At each round, the network knows which units already have an order. The units spread out instead of all going to the same place.
- **Training:** PyTorch on GPU, about 12 passes over the data. The network learns both to reproduce the moves of the top players and to predict the winner.
- **In the bot:** 82,000 to 99,000 parameters depending on the version, with weights quantized to 7 or 8 bits to fit in the 100,000-character limit. The network is rewritten in C++ with AVX2 instructions.

**Each turn:**
1. The network proposes up to 24 plans for me: the first one is the most likely, the others are sampled from its probabilities.
2. It also proposes up to 8 plans for the opponent, looking at the board from their side.
3. For each pair (my plan against an opponent plan), the simulator plays the turn exactly.
4. The network scores the resulting position with its value output.
5. A plan's score is its average over the opponent's plans.
6. Weak plans are dropped along the way, to spend more time on the good ones.

**Result.** First in Gold, then promoted to Legend on November 7, the day the league opened.

### Version 4 — More data and better candidates (final version)

What changed:
- **Data:** games of the top 100 instead of the top 30.
- **Cheaper sampled plans:** built in 4 rounds instead of 6, so more plans fit in the time budget.
- **Attack plans added:** my triangles hit the biggest neighboring enemy stack. Before, against strong players, the bot almost never attacked: not a single attack in 22 of 27 lost games against the top 20.
- **A "maximum threat" reply for the opponent:** each enemy triangle attacks my biggest neighboring stack, to check that my plan holds.
- **A more careful score:** 30% worst case and 70% average over the opponent's plans.
- **A safety margin:** 36 ms per turn instead of 40, after a timeout in a test game.

## What didn't work

- **A network that sees the whole board, twice as fast (tested, never submitted).** At each layer, every house also receives the average of all houses. The network code was rewritten (AVX2 and FMA, weights stored by columns, 4 houses at a time, a first decoding round shared by all plans), with decisions identical to the bit. A plan took 0.95 ms instead of 1.84 ms on CodinGame servers, and the bot evaluated 60 to 70 positions per turn instead of 18. Locally, the faster version won 800 of 1,200 games against the same version at normal speed. But in test games against the real top players, it did no better than version 4, so I kept version 4.
- **Reinforcement learning (PPO).** The bot played against itself and against the classic search, and its move probabilities were corrected according to wins. It helped a little at normal speed, but not at all once the network was sped up.
- **A bigger network, with 5 layers:** same accuracy, but slower, so weaker.
- **More data:** imitation plateaus at about 58.5% of moves predicted.
- **Automatic tuning of the classic evaluation (SPSA):** it beat my reference bot, but not my other versions.
- **Looking 2 turns ahead in the classic search,** with a fast policy for both sides: no gain at real speed.
- **Other algorithm families, compared early in a local tournament:** a Smitsimax-style search won only 8 to 18% of its games against the local search, because the units did not coordinate. Genetic co-evolution won 1%.

## Where the limit is

I replayed my 449 ranked games to find the cause of my losses:
- **The bot sees the opponent coming:** it had foreseen 85% of the opponent's captures, 96% of my failed captures and 98% of the attacks it suffered, in wins as in losses.
- **The network's value predicts the winner well,** better than the difference in units, and my last 4 networks are the same on this point.
- **Even the best of my 24 candidate plans sends only 72% of the units where a top-10 player sends them,** and more plans barely change it.

So the limit is the quality of the imitation, not the search or the speed. Going further would need a search over several turns, with a much faster network trained by reinforcement for days.

## Tooling

- **An exact C++ simulator of the game,** checked against the official referee: 0 differences over about 350 games and 45,000 turns. It also replays real games exactly.
- **A local arena running games in parallel:** thousands of games per test, with a virtual clock calibrated on CodinGame, so results don't depend on the load of the machine.
- **Profiling with the CPU cycle counter,** to know where the time goes before optimizing.
- **Exactness checks:** every fast version (simulation, potential, network) is compared to its reference version, plan by plan.
- **A bot generator** that fits the network into 100,000 characters: weight encoding, code compression while checking that the machine code stays the same, removal of unused code.
- **Automated CodinGame test games,** with the same opponents and the same maps for each version, compared game by game.
- **Game analysis tools:** triangles taken per turn, losses, attacks, phase by phase.

## What I learned

- **A local gain doesn't guarantee an online gain.** My test opponents were bots from the same family. Real games, played at the same time against the same opponents, decide.
- **The game is decided between turns 6 and 15,** in the race for triangles. The opening is even.
- **Always judge against several different opponents.** A setting can beat one bot and lose to another.
- **A guide that predicts better doesn't always guide better.** Only games decide.
- **Speed up only without changing the decisions.** An approximate shortcut can lose more than it gains.
