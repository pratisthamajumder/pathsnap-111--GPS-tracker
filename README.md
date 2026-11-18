<<<<<<< HEAD
# pathsnap-111--GPS-tracker
=======
# PathSnap — Team 111
Map-matching and route reconstruction from noisy GPS: candidate classifier + **learned-transition HMM**.

## What is different about our approach
Most map-matchers use "nearest road + Newson–Krumm HMM". Ours has three extra pieces:

1. **Self-calibrating noise model.** Each trip estimates its own GPS noise scale σ from the GPS and the map only (σ = 1.4826 × median nearest-arc distance, a MAD-style estimator). Distance features are also given in units of σ (`dist_z`, `ctx_dist_z`), so one classifier works for both clean and badly offset trips. A **temporal context window** (±3 observations: mean/max distance to the same arc, heading agreement) lets the classifier see how a whole stretch of track fits each arc, not just one point.
2. **Hybrid learned-transition HMM.** A logistic transition model is trained on the reference transitions of half the training trips. Its features are |route distance − GPS distance|, their ratio, same-arc, U-turn, moving backwards on an arc, turn angle, time gap, and unreachable. We compared three decoders: *classic* (Newson–Krumm exponential), *learned* (logistic model only) and *hybrid* (classic distance prior + learned correction). Decoder weights were tuned on held-out training trips using out-of-fold classifier scores. **Hybrid won the tuning and is deployed.** On student-test, point accuracy was classifier-only 89.2% → classic 94.7% → learned 94.2% → hybrid 95.5% (`results/hmm_comparison.csv`).
3. **Confidence + stitched route.** Forward–backward gives a posterior confidence for each observation (the UI can colour points by it). Gaps between non-adjacent decoded arcs are filled with shortest paths, so the output is one continuous route you could actually drive.

## Repository layout
```
src/pathsnap/      data.py (loading, projection, spatial index) · labels.py (route→per-point labels)
                   features.py (candidates + features) · hmm.py (Viterbi, forward-backward, stitching)
                   metrics.py · pipeline.py (inference used by the API)
scripts/           01_eda.py 02_build_dataset.py 03_train_compare.py 04_bayes_opt.py
                   05_hmm.py 06_maps.py 07_plots.py api_client.py common.py
api/main.py        FastAPI backend (Laptop 1)
ui/index.html      evaluator UI (Laptop 2), Leaflet
models/            final_model.joblib, transition_model.joblib, model_config.json, hmm_config.json
results/           every CSV / JSON / log produced by the scripts
figures/           every plot used in the report     maps/  Folium HTML maps
data/raw/          train/ (80) · student_test/ (10) · evaluator_test/ (put the 10 evaluator trips here)
```

## Install
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```
The saved `.joblib` models were built with Python 3.13 and scikit-learn 1.9.1. If your scikit-learn version differs, re-create the artifacts with `./run_all.sh`, or run scripts 02 → 04 → 05 (about 30 min).

## Reproduce the experiments
```bash
./run_all.sh          # or run scripts/0*.py one by one, in the order listed in run_all.sh
```
Seeds are fixed (`SEED = 111`). The evaluator partition is never read by any training or tuning script.

## Run the two-laptop demo
**Laptop 1 (API + model)**
```bash
# copy the evaluator trips to data/raw/evaluator_test/<trip_id>/<trip_id>.{track,nodes,arcs,route}
uvicorn api.main:app --host 0.0.0.0 --port 8000
# optional: PATHSNAP_EVAL_DIR=/path/to/eval uvicorn api.main:app --host 0.0.0.0 --port 8000
```
Find Laptop 1's IP (`ipconfig` / `ip a`) and allow port 8000 through its firewall. Each request is logged in this terminal (`REQUEST …` and `PREDICT trip=… acc=…`). Those log lines are the "API logs on the receiving laptop".

**Laptop 2 (UI)**
```bash
cd ui && python -m http.server 8080
# open http://localhost:8080, enter http://<laptop1-ip>:8000, click "Connect & load trips", pick a trip
```
Command-line check from Laptop 2: `python scripts/api_client.py http://<laptop1-ip>:8000`

### API
| Method | Path | Description |
|---|---|---|
| GET | `/health` | status, model, decoder |
| GET | `/trips` | evaluation trip ids found in `PATHSNAP_EVAL_DIR` |
| POST | `/predict` `{"trip_id": "..."}` | full pipeline; returns GPS, per-point arc + confidence, stitched route polyline, metrics |
| GET | `/predict/{trip_id}` | same as POST |

Reference routes are read only in `PathSnapPredictor.evaluate()`, on the server, after prediction. Only aggregate metrics are returned. No reference geometry or labels are sent to the UI.

## Pipeline summary
1. **Clean**: drop non-finite / out-of-range coordinates and non-increasing timestamps (none found in train / student_test, but handled).
2. **Project**: per-trip local equirectangular projection (metres), centred on the trip mean.
3. **Reference labels**: monotonic DP alignment of GPS points to the ordered `.route` arcs (`labels.py`). Trips 33 and 54 have most points more than 100 m from their route, so they are excluded from training as unreliable labels.
4. **Candidates**: arcs within 50 m (KD-tree over points sampled along arcs), at most 8 per point. If none are within 50 m, fall back to the 3 nearest arcs.
5. **Features** (22): distance, rank, gap to nearest, heading difference, speed, arc length, projection position, two-way flag, context-window distances and headings, time gap, node degrees, noise-normalised distances.
6. **Classifier**: 8 model families compared with 5-fold GroupKFold (groups = trips). HistGradientBoosting is selected and tuned with our own GP + Expected-Improvement Bayesian optimiser.
7. **Decoder**: hybrid learned-transition HMM (Viterbi), forward–backward confidence, shortest-path stitching.
>>>>>>> 94f3a08 (Add Bayesian optimisation and HMM notebooks and trained models)
