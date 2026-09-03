# CAAST

**Convective AFD Analog Semantic Tool.** A retrieval model that reads what a National
Weather Service forecaster wrote this hour and finds the historical forecast discussions
that read most like it, then reports what actually happened on those days.

The premise is that a forecaster's prose carries information the model guidance does not.
When someone writes "instability is marginal but the shear profile is impressive," they are
compressing a judgment that no single parameter captures. CAAST treats that prose as the
query and 5.2 million archived discussion sections as the corpus.

---

## What it does

Every hour, for each of 122 Weather Forecast Offices:

1. Fetches newly issued Area Forecast Discussions from the Iowa Environmental Mesonet.
2. Extracts the Long Term section, falling back to Extended or Discussion when absent.
3. Embeds it with a fine-tuned 335-million-parameter sentence-transformer encoder.
4. Searches that office's own historical corpus for the ten nearest neighbors by cosine
   similarity.
5. Looks up what verified on each of those historical days: tornado, severe wind, severe
   hail, or nothing.
6. Compares the severe fraction among the analogs against **that office's own base rate**
   and emits a signal level.

Output is written as JSON for a static frontend and committed back to this repository, so
the published data and the code that produced it share a history.

## Why per-office baselining matters

A severe fraction of 0.4 among ten analogs means something very different in Norman than in
Caribou. Scoring against a global average would make every plains office look permanently
dangerous and every northeastern office look permanently quiet, which is a climatology
detector rather than a forecast signal. Every threshold in CAAST is calibrated against the
issuing office's own base rate, so the number reports departure from normal *for that
office* instead of restating where severe weather is common.

## Signal levels

| Level | Meaning |
|---|---|
| `HIGH` | Analog severe fraction exceeds the office's high threshold above its base rate |
| `MODERATE` | Above the moderate threshold |
| `LOW` | Above the low threshold |
| `QUIET` | At or below the office's normal rate |
| `INSUFFICIENT_MATCH` | Mean similarity of the top ten analogs fell below 0.70 |

`INSUFFICIENT_MATCH` is deliberate and it fires often. If the nearest historical discussions
are not actually similar, the analogs are noise and the honest output is a refusal rather
than a number. A retrieval system that always answers is a retrieval system you cannot
trust when it matters.

## Evaluation

The model was verified across 91 forecast offices with enough labeled history to score:

| Metric | Value |
|---|---|
| Precision | 67.6% |
| Mean lift over per-office climatology | +15.9 points |
| Offices evaluated | 91 |
| Corpus | 5.2 million discussion sections |
| Training pairs | 1.44 million |

Skill decay was measured day by day across the forecast lead range rather than reported as
a single aggregate. The lift figure is the one that matters: precision alone would look
respectable in any office where severe weather is common, and the interesting question is
whether the model beats simply knowing the office and the season.

## Training

The encoder was fine-tuned end to end from a base checkpoint through domain-adaptive
pretraining on the discussion corpus, followed by contrastive learning with hard-negative
mining on 1.44 million pairs. Hard negatives matter more than usual here: two discussions
from the same office in the same month share enormous surface vocabulary while describing
completely different outcomes, so a model trained on random negatives learns to retrieve
regional writing style instead of meteorological similarity.

Training was run on [Runpod](https://www.runpod.io) GPU instances.

## Repository layout

```
scripts/hourly_update.py      the production job, end to end
.github/workflows/            hourly GitHub Actions schedule
web/data/signals.json         current signal level per office
web/data/analogs/<WFO>.json   the ten analogs behind each office's current signal
web/data/last_run.json        watermark for incremental fetching
```

The model weights, the embedded corpus, the label set and the per-office thresholds live in
Cloudflare R2 rather than in git, because they are several gigabytes. The hourly job pulls
and caches them on first run. **This repository contains the inference and production side.**
The training and corpus-construction code is not published here.

## Running it

```bash
pip install sentence-transformers boto3 requests numpy

export R2_ACCESS_KEY_ID=...
export R2_SECRET_ACCESS_KEY=...
export R2_ENDPOINT=...
export R2_BUCKET=caast-corpus

python scripts/hourly_update.py
```

The job is incremental. It reads `web/data/last_run.json`, fetches only discussions issued
since that timestamp, and falls back to a 24 hour lookback on a cold start. Model and
per-office embeddings are cached under `.caast_cache/`, so the first run is slow and every
run after it is not.

In production it runs under GitHub Actions at 15 minutes past each hour, offset off the top
of the hour to avoid the Iowa Environmental Mesonet's peak load, and commits any changed
output back to the repository.

## Limitations

- **It reads forecasters, not the atmosphere.** If the discussion does not describe the
  convective setup, there is nothing to retrieve against. Quiet-day prose produces quiet-day
  analogs regardless of what the models are showing.
- **Analog reasoning is not causal.** Similar language on similar days is a correlation, and
  the model has no physical understanding of why any of it happened.
- **Corpus depth varies by office.** Offices with shorter or sparser archives produce
  `INSUFFICIENT_MATCH` more often, which is the correct behavior but reduces coverage.
- **Labels come from verified storm reports**, which carry their own well-documented
  population and reporting biases.
- This is a research tool. It is not an operational product and nothing here should be used
  to make a warning decision.

## Data sources

Area Forecast Discussions and archived text via the
[Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/). Severe weather labels
derive from verified storm reports.

## Author

Alex Cooke. Operational meteorologist and editor. [alexcooke.co](https://alexcooke.co)
