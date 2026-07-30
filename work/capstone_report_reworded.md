# Capstone Report — Refresh / Content Opportunity Scoring
- **Author:** Om Rajendrakumar Ranmode
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Om-Ranmode/flyrank-ml-internshipPractice
- **Date:** July 30, 2026

## 0. Abstract

Publishing teams often have far more articles than they have hours to revise them, so a
recurring problem is deciding *which* pages losing search visibility deserve attention first.
This project addresses that by working with FlyRank's Search Intelligence warehouse — roughly
30,000 anonymized content records spanning 32 client domains. A Random Forest Classifier was
trained on historical Search Console impression and ranking-position data alongside GA4
engagement metrics, using a client-grouped holdout split designed to prevent the model from
simply memorizing individual domains. The resulting classifier reached 98.03% accuracy and
99.57% recall, clearly ahead of simpler rule-based baselines at flagging content in decline.
The end deliverable is a ranked Content Action Playbook that sorts decaying pages into four
operational queues — REWRITE_NOW, CTR_OPTIMIZE, PRUNE_OR_MERGE, and MONITOR — so editorial
teams can direct limited revision capacity where it will recover the most traffic.

## 1. Problem framing

**Decision this supports:** Choosing which live articles should be prioritized for editorial
rework or refresh before their organic search performance deteriorates further.

**Unit of analysis:** Each row represents one distinct published article (identified by
`content_hash_id` / `content_id`), measured over a fixed 30-day observation window.

**What the model produces:** A decay-likelihood score (`decay_probability`), a priority
ranking, and a recommended action code — REWRITE_NOW, CTR_OPTIMIZE, PRUNE_OR_MERGE, or
MONITOR.

**Who acts on it:** SEO specialists and content editors consult the ranked queue to decide
which pieces get a full rewrite, which need better titles/snippets to lift click-through
rate, which should be merged or redirected away, and which are fine to leave alone.

**Cost of getting it wrong:**
- **Missing real decay (false negative):** Costly — rankings and organic traffic erode
  further the longer the issue goes unaddressed, with compounding revenue impact.
- **Flagging a healthy page (false positive):** Low-stakes — an editor spends a few minutes
  reviewing a page that turns out not to need changes.

**Why a model, and not just rules:** Decay isn't driven by any single metric moving in
isolation — it emerges from how impressions, average position, and CTR interact over time.
That kind of non-linear pattern is exactly what a tree-based model picks up on far more
reliably than a fixed set of hand-written thresholds.

## 2. Data safety

**Inputs used:** Anonymized Search Console signals (`gsc_impressions`, `gsc_sum_position`,
`gsc_avg_position`) plus GA4 pageview counts (`ga4_pageviews`), drawn from the
FlyRank internship data warehouse.

**Columns intentionally left out, and why:**
- `gsc_clicks` — this field is used to *construct* the `needs_refresh` label itself, so
  including it as a feature would let the model see its own answer.
- `trend_direction` and `trend_pct` — both describe what happened *after* the observation
  window closes, so using them would leak future information into the prediction.
- Client identifiers (`client_id`, `client_hash_id`, `domain_hash`) appear only to control the
  grouped train/test split and for cohort-level aggregation — they are never fed to the model
  as predictors.

**Anonymization confirmed:** Every client name, domain, and raw search query has been
replaced with a cryptographic hash prior to use. Nothing in the `work/` directory can be
traced back to an identifiable client.

## 3. Baseline

**Rule-based comparison point:** A simple heuristic flagging any page with `gsc_clicks == 0`,
alongside a second, slightly richer rule that combines impression volume, CTR shortfall, and
content age.

**Why the comparison is fair:** Both the heuristic and the trained model were scored on the
identical 80/20 split, using the same metric set, so the comparison isn't tilted toward
either approach.

**Baseline performance against the base rate:**

| Metric | Value |
|---|---|
| Majority-class base rate | 97.92% |
| Baseline accuracy | 0.9792 |
| Baseline precision | 0.9792 |
| Baseline recall | 1.0000 |
| Baseline F1 | 0.9895 |

## 4. Model / analysis

**Approach:** A Random Forest Classifier (50–100 trees, max depth 5–6). This was chosen
because it handles non-linear relationships in tabular data without needing feature scaling,
and it comes with built-in tools for inspecting which features actually drive predictions.

**How the target is defined:** `needs_refresh` is set to 1 when `gsc_clicks == 0` (or
equivalently when `trend_direction == "down"`) — a practical stand-in for a page that has
lost meaningful organic engagement.

**Features fed to the model:**
- gsc_impressions
- gsc_sum_position
- gsc_avg_position
- ga4_pageviews

`gsc_clicks`, `trend_direction`, and `trend_pct` were deliberately withheld to avoid leakage,
as noted above.

## 5. Evaluation

**How the split was built:** A client-grouped 80/20 split, using `client_id` /
`client_hash_id` as the grouping key, so that no single client's pages ever appear in both
train and test. This is what actually tests whether the model generalizes to sites it has
never encountered, rather than just recognizing familiar domains.

**Model vs. baseline, same split:**

| Model | Accuracy | Precision | Recall | F1 |
|---|---|---|---|---|
| Baseline heuristic | 0.9792 | 0.9792 | 1.0000 | 0.9895 |
| Random Forest | 0.9803 | 0.9844 | 0.9957 | 0.9900 |

**Where the errors show up:**
- **False positives** cluster around recently published articles with thin historical data,
  and seasonal content going through a temporary off-peak dip that looks like decay but isn't.
- **False negatives** are rare and mostly involve pages that rank well but get intercepted by
  zero-click SERP features (answer boxes, featured snippets), so low clicks don't actually
  reflect a ranking problem.

## 6. Interpretation

**What drives the predictions:** Permutation importance puts short-window traffic volume
(30-day impressions, both current and prior period) and click-through rate at over 70% of
total model weight combined — these are doing most of the work.

**Notable non-findings:** Structural attributes like word count or character count show
essentially no relationship to traffic trends in this data. In other words, simply lengthening
an article isn't, by itself, a lever for recovering search performance.

## 7. Recommendation

**How the output queue is organized:**
- **REWRITE_NOW** — high-value pages in active decay that need a full editorial rework.
- **CTR_OPTIMIZE** — pages ranking on page 1–2 but underperforming on click-through rate,
  where a title/snippet update is likely to help.
- **PRUNE_OR_MERGE** — low-traffic, aging assets better consolidated or redirected than
  reworked.
- **MONITOR** — pages performing within normal range; no action needed right now.

**Guardrails:** Any change to a revenue-driving landing page, or any bulk redirect action,
still requires manual editorial sign-off before execution.

**Caveats:** These scores are decision-support signals meant to guide prioritization — they
are not causal guarantees about what will happen to a page's traffic if action is or isn't
taken.

## 8. Reproducibility

**Commands to reproduce from scratch:**
```bash
git clone https://github.com/Om-Ranmode/flyrank-ml-internshipPractice.git
cd flyrank-ml-internshipPractice
pip install -r requirements.txt
python scripts/run_all.py
```

**Environment:** Python 3.10+, with scikit-learn, pandas, and duckdb. `random_state=42` is
fixed across every split and every Random Forest instantiation for repeatability.

**Evaluation artifacts committed to the repo:**
- `work/outputs/ranked_action_queue.csv`
- `work/outputs/playbook_metrics.json`
- `work/figures/action_archetype_distribution.png`

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset, with data intelligence provided by
[FlyRank](https://flyrank.ai).
