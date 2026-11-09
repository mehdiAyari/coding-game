# Troll Farm — CodinGame Spring Challenge 2026

**English** | [Français](README.fr.md)

| | |
|---|---|
| **Contest** | [CodinGame Spring Challenge 2026](https://www.codingame.com/contests/spring-challenge-2026-troll-farm), May 12–25, 2026 |
| **Game** | Troll Farm: two players control trolls that harvest, chop and mine on a grid, simultaneous turns |
| **Result** | Legend league, the top league |
| **Final bot** | Hand-written rules and economic estimates, in four phases, with two trolls |
| **Stack** | Java |

## The game

- Each player starts with a shack and one troll, on a grid of grass, water, rock, iron and trees (8 to 11 rows, twice as many columns).
- Trolls harvest fruit from trees, chop trees for wood, mine iron, and bring everything back to their shack.
- Trees (plum, lemon, apple, banana) grow up to size 4, then produce up to 3 fruits. They grow faster next to water. A troll can plant a tree with a fruit it carries.
- Resources pay for new trolls. Each of the 4 attributes (speed, carrying capacity, harvest power, chop power) costs a given fruit or iron, and its cost is the number of trolls plus the attribute squared.
- Score: each fruit in the shack is worth 1 point, each piece of wood 4 points. Iron is worth nothing.
- Both players play at the same time, with 50 ms per turn, for at most 300 turns.

## The final bot in brief

The bot goes through four phases (detailed in version 4 below):

1. **A single troll:** it works out which second troll will bring the most wood, and gathers what it costs. It plays the whole game with these two trolls.
2. **Two trolls:** wood is worth 4 times a fruit, so they chop, starting with the lemon trees near the opponent's shack, then the other trees through a global assignment.
3. **No tree left:** the slow troll turns the shack's fruits into wood (1 point becomes 4), while the fast troll heads to the opponent's shack.
4. **No fruit left:** both trolls stay next to the opponent's shack and chop whatever the opponent plants.

There is no search and no simulation of the whole game: every decision comes from hand-written rules and estimates.

## Timeline

| Version | Date | Main idea | League |
|---|---|---|---|
| 1 | May 12–13 | Simple rules: harvest, bring back, then chop | Wood → Bronze |
| 2 | May 14–15 | Choices by points per turn, a banana farm next to the shack | Bronze → Silver |
| 3 | May 17 | "Chop first" in phases, tree growth and races simulated | Silver → Gold |
| 4 | May 21–22 | Four phases: the second troll chosen by expected wood, the opponent's lemon trees first | Gold → Legend (final version) |

## Bot versions

### Version 1 — Simple rules

- **The first bot (fruit only, in the first league):** drop the fruit when next to the shack; harvest when on a tree with fruit and room left; otherwise go back to the shack when carrying fruit, or go to the nearest tree with fruit.
- **Then, with the full rules:** training a troll, mining, chopping. Trees are scored by points per turn spent.
- **Sticky targets:** each troll keeps its target from one turn to the next, and only switches if a new target is at least 30% better. This stops the zigzagging.

### Version 2 — Choices by points per turn

- **A banana farm next to the shack:** the slowest troll picks a banana from the shack, plants it on a cell next to the shack, chops the tree and drops the wood. Bananas have the shortest cycle: with 3 health points, the tree is chopped before it grows.
- **The farm cell:** not next to water when possible (the tree would grow faster and lengthen the cycle), and not on a traffic route. The bot simulates a tree planted on each candidate cell and measures how much it lengthens the paths to the shack.
- **Points per turn:** the slow troll compares three options by their return: farm, chop a tree of the map, or harvest one. The real cost of a farm cycle is simulated turn by turn, because a tree can grow back during the chop.
- **My own pathfinding:** a BFS that goes around trees and other trolls, written after a troll got stuck for about 45 turns.
- **No more conflicts:** trolls reserve their targets, so a troll can no longer take a tree another one is already chopping. And if the farmer blocks another troll, it steps aside.

### Version 3 — "Chop first", in phases

- **Training:** the bot first aims at a minimum troll (speed 2, capacity 2, chop 2, no harvest), then at the best combination it can afford now or after a short wait. The waiting time is set by the slowest resource to gather, since the others are gathered meanwhile. Once chosen, the target is locked, unless a strictly stronger one becomes possible.
- **Income trees:** plum and lemon trees planted near the shack produce fruit in the background, unless a tree of that type already grows close by.
- **Chop the whole map:** each tree is scored by wood per turn. The bot simulates its growth during the trip and during the chop, and the opponent's trolls already chopping it. A tree that the opponent will finish first is skipped.
- **A global assignment** of trees to trolls (the best troll–tree pair first, then the next one), with a bonus for the previous target, instead of each troll choosing alone.
- **Fruit into wood** when no tree is left: each troll plants fruits next to the shack and chops them. Then, with no fruit left, both trolls camp next to the opponent's shack and chop what the opponent plants.

### Version 4 — Four phases, the opponent's lemon trees first (final version)

**Phase 1 — a single troll: choose the second troll, then gather.**
- **Which troll to train.** The bot tries every troll with speed 2, capacity 2 to 5, chop 1 to 5 and no harvest. With one troll in the team, each attribute costs 1 plus its value squared: plums for speed, lemons for capacity, iron for chop, and one apple for a harvest of 0.
- **How long to wait.** For each missing resource, the bot estimates the time to gather it, and the slowest resource sets the waiting time. Iron: one round trip to the nearest iron per unit. A fruit: the better of fetching it from the nearest tree of that type (a round trip, or the tree's regrowth time if longer) and relying on a tree near the shack (an existing one within 3 steps, or one to plant next to it). A troll that needs more than 5 extra units of one resource is ruled out.
- **The score:** the wood this troll would bring, that is its wood per turn (round trip at its speed over an average distance, chop time, wood carried) times the turns left before turn 100 once the wait is over.
- **Close shacks:** a troll that makes the bot wait more than 25 turns is ruled out, and more than 2 turns when the shacks are 2 steps apart or less. When they are 8 steps apart or less, the horizon is shortened by up to 30%.
- **A locked target:** the chosen troll changes only for a strictly better one, or when it can no longer be reached. If no troll fits, the bot falls back to the minimum: speed 2, capacity 2, no harvest, chop 1.
- **Gathering:** the troll compares mining and harvesting by the useful units brought back per turn of round trip, weighted by the share of the cost still missing. It keeps its target unless another one is 30% better. When nothing is worth it, it walks to the nearest needed tree that will soon bear fruit.
- **Income trees:** with plums or lemons beyond the training cost, the troll first plants a plum tree, then a lemon tree, 1 or 2 steps from the shack (next to water if possible, so that it produces faster), unless a tree of that type already grows within 3 steps. Once this is settled, if iron is still missing, harvesting is heavily penalized: iron first.

**Phase 2 — two trolls: chop, the opponent's lemon trees first.**
- **The opponent's lemon trees.** Many opponents harvested lemons to train a troll with a large carrying capacity, and I lost many games against them. So the bot ranks the lemon trees within 6 cells of the opponent's shack by their value for the opponent (fruits now, plus their production over 30 turns) times their closeness to its shack. Each tree gets the closest free troll: one troll per tree, so both trolls go when there are two such trees.
- **The other trees** are shared by the global assignment: wood per turn for each troll–tree pair (trip, chop, return), with the tree's growth and the opponent's trolls already chopping it simulated. A tree the opponent will finish first is skipped. The best pair is taken first, the previous target gets a ×1.5 bonus, and on a tie the tree closer to the opponent's shack wins.
- **No deadlock:** the fast troll decides first. The slow troll's moves go through my own BFS, around the fast troll's next position rather than its current one.

**Phase 3 — no tree left: fruit into wood.**
- The slow troll converts the fruits of the shack: it picks one, banana first, then plum, lemon and apple (the quickest trees to chop first), plants it on a cell next to the shack, chops the tree and brings back the wood. A fruit worth 1 point becomes wood worth 4.
- Meanwhile, the fast troll already leaves for the opponent's shack (phase 4).

**Phase 4 — no fruit left: at the opponent's shack.**
- Both trolls chop the nearest tree they can reach before the opponent finishes it (by then, the trees the opponent plants), and otherwise wait on a cell next to the opponent's shack.

**Result:** promoted to Legend.

## What didn't work

- **Farming slow fruits next to the shack** (plum, lemon, apple): the tree grows back faster than it is chopped (an apple has 11 health points and a 9-turn cooldown), it stays on the farm cell and blocks the way, and slow farming leaves time for the opponent to score. Only bananas were kept.
- **Choosing the farm cell by its distance to the edge:** it picked cells on critical corridors, once even the only passage to the shack. Replaced by the measure of how much a tree there lengthens the paths.
- **A minimum return threshold:** it froze the slow troll at the end of the game, about 14 turns idle while the opponent kept scoring. Now the troll acts as soon as any option pays.
- **Each troll choosing its tree alone:** a troll could take the tree another one was already chopping. Replaced by reservations, then by the global assignment.

## Where the limit is

No measurement from this contest is left, only the submitted code. What the code shows:
- **No simulation of the whole game and no search:** every decision comes from rules and estimates (average tree health and size, for example). The opponent is only seen through simple signals: its trolls already chopping a tree, and where its shack is.
- **The trolls decide one after the other** (the fast one, then the slow one), with a global assignment only for the trees.
- **Two trolls for the whole game:** the bot trains a single extra troll.
- **The natural next step** would be an exact simulator of the game, to test these rules on many games and to search over the joint actions of the trolls.

## Tooling

- **A turn report in the debug output:** inventories, phase, the decision of each troll and its best candidates.
- **Prediction checks:** the bot predicted its inventory for the next turn and compared it with the real one.
- **Blockage detection:** a troll that wanted to move but did not is detected on the next turn.
- **Distances by BFS** on the real map, around water, rocks and iron, instead of straight-line estimates.

## What I learned

- **Wood is worth 4 times a fruit:** the best use of a fruit is often to plant it and chop the tree.
- **Simulate time, not just distances:** a tree grows during the trip and during the chop, and an apple can grow back faster than it is chopped.
- **Coordinate the units:** reserving targets and assigning them globally stops trolls from stealing each other's work.
- **A threshold that is too strict can freeze a unit:** acting a little beats waiting.
- **Keep a target unless a new one is clearly better** (30% here): it stops the zigzagging.
