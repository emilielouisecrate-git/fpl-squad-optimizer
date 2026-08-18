# FPL Squad Optimizer: From Rolling Form to an Optimal £100m Team

An end-to-end pipeline that predicts Fantasy Premier League points from player form,
then uses integer linear programming to build the mathematically optimal 15-player
squad under real FPL constraints (£100m budget, position quotas, max 3 players per
club) — applied to the actual, unplayed **2026-27 Gameweek 1** (kickoff 21 August 2026).

Built in R, from raw data to optimizer, gameweek predictions checkable against reality
once the season starts.

## The pipeline

1. **Rolling-form feature engineering** — for every player-gameweek in the complete
   2025-26 season (~29,700 rows), build 3-gameweek rolling averages of points, minutes,
   ICT index, BPS, and expected goal involvement — using only information available
   *before* that gameweek (no leakage: every feature is built with `lag()`, never
   looking at the target gameweek's own stats)
2. **Model training** — a random forest predicting next-gameweek points from rolling
   form, trained on GW1–28, validated on the held-out GW29–37 (temporal split, never
   trained on the future)
3. **Live application** — each returning player's *most recent* form (end of 2025-26)
   fed into the trained model to predict points for the real, upcoming 2026-27 GW1
4. **Squad optimization** — an integer linear program (`lpSolve`) selects the 15 players
   that maximise total predicted points subject to budget, position, and club constraints

## Results: does the model actually help?

Honestly — barely, and that's a real finding worth reporting rather than hiding.

| Model | Test MAE (GW29-37) |
|---|---|
| Baseline (rolling-3 average) | 1.033 |
| Random Forest (mtry=1, default) | **1.033** |
| Random Forest (mtry=3) | 1.057 (worse — mild overfitting) |

The random forest essentially ties the naive "just use recent form" baseline. Increasing
`mtry` (giving the forest more features to consider per split) made things *worse*,
which rules out under-fitting as the cause — the ceiling here appears to be genuine
noise in FPL points themselves (a hat-trick, a red card, or a bonus-point rounding
decision can swing a result in ways no rolling average can anticipate), not a tunable
hyperparameter.

**Feature importance tells a more interesting story than the MAE does:**

| Feature | %IncMSE (importance) |
|---|---|
| `bps_roll3` | 18.9 |
| `minutes_roll3` | 15.8 |
| `expected_goal_involvements_roll3` | 11.9 |
| `ict_index_roll3` | 11.2 |
| `total_points_roll3` | 5.7 |

Recent **BPS** (the underlying bonus-points-system score) is the single strongest
predictor — more than raw points scored. That makes sense on reflection: BPS is
itself a composite measure of a player's overall match involvement, so it captures
more signal than points alone, which can be lumpy (one big haul vs. steady returns).
Past points scored being the *weakest* of the five features was a genuine surprise
going in — worth stating honestly rather than pretending it was expected.

## The actual output: an optimal GW1 2026-27 squad

Solved via integer linear programming — 450 candidate players (this season's squads,
matched against last season's form), maximising predicted points subject to:
£100m budget, exactly 2 GK / 5 DEF / 5 MID / 3 FWD, and no more than 3 players from
any single club.

| Pos | Player | Price | Predicted GW1 pts |
|---|---|---|---|
| GK | David Raya Martín | £6.0m | 5.68 |
| GK | Emiliano Martínez Romero | £5.0m | 4.41 |
| DEF | Adrien Truffert | £5.5m | 4.90 |
| DEF | Marc Guéhi | £6.0m | 4.58 |
| DEF | Riccardo Calafiori | £5.5m | 4.47 |
| DEF | James Tarkowski | £6.0m | 4.44 |
| DEF | Nathan Collins | £5.5m | 4.36 |
| MID | Saša Lukić | £5.0m | 5.88 |
| MID | Enzo Le Fée | £6.0m | 5.43 |
| MID | Sandro Tonali | £5.5m | 5.41 |
| MID | Jack Hinshelwood | £6.0m | 5.18 |
| MID | Kobbie Mainoo | £5.5m | 4.58 |
| FWD | Erling Haaland | £15.5m | 7.11 |
| FWD | Omar Marmoush | £7.0m | 4.40 |
| FWD | Igor Jesus | £6.0m | 3.90 |

**Total cost: £96.0m of £100m** (£4m left unused — the position and club-limit
constraints were the real binding pressure, not the budget). **Total predicted GW1
points: 74.74.**

This table was generated before a ball was kicked in the 2026-27 season — a genuine,
checkable prediction, not hindsight.

## Data

- Historical training data and this season's squad/price lists: [vaastav/Fantasy-Premier-League](https://github.com/vaastav/Fantasy-Premier-League),
  a community-maintained mirror of the official FPL API

## Repo structure

```
├── fpl_squad_optimizer.ipynb   # full R notebook: features, model, optimizer
└── data/
    └── optimal_gw1_squad.csv   # the final selected squad
```

## Limitations and honest caveats

- **The model barely beats the naive baseline.** This is stated plainly above rather
  than buried — FPL points have a large irreducible noise component that rolling-form
  features can't capture
- **Merging last season's players onto this season's squads dropped ~46% of players**
  (838 → 450) — mostly genuine relegation and summer transfers (verified by spot-checking:
  e.g. Leandro Trossard moved to Beşiktaş), not a data bug, but it does mean any player
  who transferred didn't carry over their prior form into this prediction
- **No fixture-difficulty features** — the model doesn't know who GW1 opponents actually
  are, only recent general form. A natural next step would be incorporating opponent
  strength
- **Squad optimizer picks a fantasy-ideal 15, not necessarily a valid starting XI** —
  formation constraints (e.g. minimum starting defenders) aren't enforced on top of the
  squad-level position quotas
