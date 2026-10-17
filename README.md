# [Team Name]_IBM Z Datathon 2026 Project
## Referee Decision Triage on IBM LinuxONE

**Theme:** Real-Time AI for Critical Decisions

Referee assessors review officials after every match, but there are far too many incidents to rewatch them all. Our tool learns how the league typically calls each kind of incident, scores how surprising every real decision was, and ranks the most unusual calls first, so assessors know where to look. It runs in real time on IBM LinuxONE, and every decision it scores is written to a tamper-evident audit log.

The tool does **not** say a referee was wrong. A flag means "this call is unusual for its context, a human should look at it."

## How it works

```
StatsBomb event data ─► feature engineering ─► XGBoost classifier (LinuxONE)
                                                      │
                          P(no foul, foul, yellow, red) for each incident
                                                      │
                     surprise score vs. the referee's actual call
                                                      │
          ranked review list ─ written explanation ─ hash-chained audit log ─ dashboard
```

1. **Data.** StatsBomb Open Data, Premier League 2015/16: all 380 matches, 24,957 incidents.
   Labels: `no_foul` (15,445 clean tackles), `foul` (8,443), `yellow` (1,040; second yellows count as yellow because the foul itself was yellow-level), `red` (29).
   Fixed stratified 80/20 train/test split, shared by the whole team.
2. **Model.** XGBoost multiclass on context features: minute, period, pitch location, home/away, player position, play pattern, and fouls committed earlier in the match by the player and the team. No class weights, so the probabilities stay honest.
3. **Surprise score.** `-log( P(referee's actual call) / base rate of that call )`. Dividing by the base rate means a red card isn't "surprising" just because reds are rare, only if it was less likely than usual in that context.
4. **Explanations.** For each flagged incident, a plain-English note: what was called, what the model expected, how unlikely the call was, and what to check in the footage.
5. **Audit log.** Every scored incident is hashed together with the previous entry's hash (SHA-256 chain). Editing any past entry breaks the chain from that point, and verification reports exactly which entry was changed.
6. **Dashboard.** Jupyter + ipywidgets on LinuxONE: filters, ranked review list, per-incident probabilities, pitch map, and a live match replay.

## Results (test set, 4,992 incidents the model never saw)

| Metric | Model | Baseline (class frequencies) |
|---|---|---|
| Log loss | **0.723** | 0.804 |
| Card vs. no-card AUC (fouls only) | **0.64** | 0.50 |
| No-foul F1 | 0.76 | |

The probabilities carry real information, which is what the ranking relies on. Distinguishing cards is hard from event data alone (see Limitations).

**Live replay** (Newcastle vs Tottenham, retrained without this match): 61 incidents scored one by one at **~4 ms each**, logged to the audit chain in **~0.03 ms each**, 5 flagged for review, including Mitrović's red card at 64'.

**Throughput:** 24,957 incidents scored in 0.16 s (~158,000 incidents/sec).

## Why IBM Z / LinuxONE

- **Real time:** every incident is scored and logged in milliseconds, inside the same system that stores the decisions.
- **Security and integrity:** the hash-chained audit log makes any later edit to a decision record detectable, the kind of integrity guarantee mainframes are built around.
- **Everything runs on the platform:** data, model training, scoring, audit log and dashboard all live on our LinuxONE Community Cloud instance (Ubuntu 24.04, s390x, Docker, IBM Jupyter Lab ML container).

## What we learned along the way

- **A data leak.** Our first model scored ~0.99 F1 on foul vs no foul. The cause: StatsBomb's `under_pressure` field is always 1 for tackles and almost never for fouls, a logging artifact. We removed it; the numbers above are the honest ones.
- **Testing our own headline.** Card rates per referee range from 8.7% to 15.0%, which looked like a story about biased referees. A chi-square test across all referees gave p = 0.15, not significant. So we built the tool around individual decisions instead of accusing referees.
- **Generative AI on the platform.** We tried writing explanations with IBM Granite (via llama.cpp with zDNN acceleration). The model available in the container (Granite 4.0, 350M, base) repeated itself and invented details, so we generate explanations directly from the model's numbers. For an assessment tool, accuracy matters more than fluency.

## Limitations

- Event data records where, when and in what context a foul happened, not the physical contact that actually decides a card. The model learns how referees typically call a context, not the correct call.
- Only 29 red cards in the season (6 in test), so red-card metrics are noisy.
- One league and one season.
- The train/test split is by incident; the live demo retrains without the demo match so it never sees that game.

## Repository

| File | What it does |
|---|---|
| `01_data_prep.ipynb` | Downloads StatsBomb data and builds the labelled dataset with the train/test split |
| `02_analysis_and_model.ipynb` | Referee analysis, model, surprise score, live replay with audit log, explanations |
| `Dashboard.ipynb` | Interactive review dashboard |

## Team

- [Name]
- [Name]
- [Name]
- Santiago Freile

## Open-source resources

- StatsBomb Open Data: https://github.com/statsbomb/open-data
- statsbombpy: https://github.com/statsbomb/statsbombpy
- pandas, NumPy, SciPy, scikit-learn, XGBoost
- matplotlib, mplsoccer, Plotly, ipywidgets
- IBM LinuxONE Community Cloud Jupyter Lab ML: https://github.com/linuxone-community-cloud/jupyter-lab-ml
- IBM Granite (via llama.cpp)
