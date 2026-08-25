# Account Takeover Detection — Behavioral Anomaly Classifier

**Author:** Martin James Ng'ang'a | Nairobi, Kenya 🇰🇪

**Status:** ⏳ Planned — Week 13

**Stack:** Sequence modeling (LSTM / gradient-boosted feature engineering) 
· Vertex AI Training · FastAPI · PostgreSQL · Langfuse · Docker · GCP

---

## The Problem

A fraud model asks: "is this transaction bad?"

It never asks the question underneath it: "is this even the real account owner logging in?"

Account takeover is a different attack entirely — a criminal who has stolen or guessed valid credentials doesn't need to trick a fraud model, because from the model's point of view, every transaction they make looks exactly like the real customer's. The fraud is upstream of the transaction, in the login itself.

This is not a hypothetical gap. Account takeover is consistently one of the most common real-world attack vectors in fintech — arguably more common than the pure transaction-level fraud most portfolios (including my own, until now) model in isolation.

**The classic signature this project targets:** a password reset, followed by a login from a new device, followed by a large withdrawal or transfer, all within a short window. Individually, none of these actions look suspicious. In sequence, at that velocity, they are one of the most reliable account-takeover patterns known in the industry.

---

## What This Solves That Transaction-Level Fraud Doesn't

| Transaction-level fraud (existing project) | Account takeover (this project) |
|---|---|
| Unit of analysis: one transaction | Unit of analysis: one session / event sequence |
| Question: is this transaction's pattern anomalous? | Question: is this the real account owner? |
| Signal: amount, time, merchant category | Signal: device change, geo-jump, login velocity, reset-to-withdrawal timing |
| Model type: single-row classification (Random Forest) | Model type: sequence-aware (session windows, time-since-last-event features) |

---

## Where This Fits

This is not a standalone model — it's designed from day one to be the second input stream into the same response infrastructure the fraud detection project already feeds:

- **SIEM correlation** — an account-takeover score arriving alongside a transaction fraud score gives a SIEM two independent, complementary signals to correlate, not one.
- **Case management** — the same analyst reviewing a flagged transaction should see a takeover risk score attached to the session, not just the transaction.
- **SOAR automated response** — a high-confidence takeover signal justifies a different, faster automated action than a fraud flag alone — forcing re-authentication or freezing the session.
- **Identity and Access Management (IAM) integration** — this is the one detection layer that plugs directly into authentication infrastructure itself (step-up MFA triggers, forced logout).
- **Feedback loop** — a confirmed takeover case (customer reports unauthorized access) is a stronger, more unambiguous label than a disputed transaction.

Two detection layers feeding one response system is closer to how real security teams actually operate than either layer alone.

The same sequence-and-velocity approach extends naturally to vishing (voice-based social engineering) — a well-documented fraud vector in Kenya's mobile money ecosystem. The call itself isn't the signal; the account action that follows it within minutes is — the same downstream behavioral pattern this project already detects, applied to a different upstream trigger.

---


## Dataset

**[Login Data Set for Risk-Based Authentication](https://www.kaggle.com/datasets/dasgroup/rba-dataset)** — 33M+ login attempts, 3.3M+ users, from a large-scale single sign-on (SSO) service in Norway (Feb 2020 – Feb 2021).

Published by Wiefling, Jørgensen, Thunem & Lo Iacono, *Pump Up Password Security! Evaluating and Enhancing Risk-Based Authentication on a Real-World Large-Scale Online Service*, ACM Transactions on Privacy and Security (2022).

Includes a genuine `Is Account Takeover` label — logins identified as account takeover by the original service's own incident response team — alongside IP/geolocation, device/browser fingerprint, login timestamp, and round-trip time (RTT) features.

**Honest disclaimer, stated by the dataset's own authors:** feature values are statistically faithful to real-world patterns but are synthetically generated — the dataset explicitly should not be used in productive intrusion-detection systems. Appropriate for research and portfolio work, which is exactly how it's used here; not a claim of raw production traffic.

Licensed CC BY 4.0.

## Build Plan

**Phase 1 — Data**
Source: a public credential-stuffing / account-takeover dataset, or synthetically generated realistic session sequences if no suitable real dataset exists — documented honestly either way. Unit of analysis: session/event sequences per account.

**Phase 2 — Feature Engineering**
Time-since-last-login, device-change flag, geo-jump distance, password-reset-to-action velocity, login time-of-day deviation from the account's own history.

**Phase 3 — Modeling**
Baseline: Random Forest / XGBoost on engineered features. Stretch: LSTM on raw session sequences, compared against baseline.

**Phase 4 — Training Infrastructure**
Vertex AI Training (GCP) — deliberate choice to build genuine training experience on a second cloud platform, distinct from SageMaker Training Jobs already used on AWS.

**Phase 5 — Serving & Observability**
FastAPI endpoint, same pattern as existing production APIs. Langfuse for any LLM-assisted explanation component; Prometheus/Grafana for core model monitoring.

**Phase 6 — Documentation**
Same standard as every other project: real numbers, honest status, "Where This Fits" from day one.

---


## Proposed Stack

Chosen deliberately, not by default — each tool solves a specific problem this project genuinely has.

| Tool | Why this one, specifically |
|------|------------------------------|
| **Polars** | 33M rows — large enough that pandas' single-threaded, in-memory model genuinely struggles. Polars' Rust-based engine handles this comfortably on a single machine, without the setup overhead of a distributed Spark session. |
| **scikit-learn** (Random Forest / XGBoost) | Proven, defensible baseline — same family as the fraud detection model. Stretch goal: an LSTM on raw session sequences, compared honestly against this baseline. |
| **SHAP** | Every prediction needs a traceable reason — which feature (geo-jump, device change, velocity) drove a specific session's score, not just a global feature importance ranking. |
| **MLflow** | Tracks experiments across baseline vs. sequence model, and across calibration methods — same tool already used for the loan portfolio CLV model, kept consistent. |
| **Evidently** | Attacker behavior evolves — new device-spoofing techniques, new IP rotation patterns. A takeover model that goes stale silently is a real risk. Evidently detects that drift. Already used in the customer segmentation project. |
| **FastAPI** | Serving layer, consistent with every other production API in this portfolio. |
| **PostgreSQL** | Live prediction audit log only — not the training data store. See *Data Flow* below for why. |
| **Prometheus + Grafana** | Live monitoring, same pattern as fraud detection. |
| **Apache Airflow** | Scheduled retraining as new labels (confirmed takeover cases) become available — closing the feedback loop. |
| **Kafka + Redis** | Phase 6 only — simulates a real-time login stream from the historical dataset, with Redis as the feature store holding each user's last known login state for instant geo-jump/velocity comparison. |
| **Vertex AI Training** | Deliberate choice to build genuine training experience on GCP, distinct from the SageMaker Training Jobs already used on AWS for the deforestation project. |
| **config.yaml** | Real thresholds (what geo-jump speed counts as "impossible travel," what calibrated score triggers an alert) need to be tunable by a security team without touching code. |
| **tests/** | Basic pytest coverage — feature engineering output shape, API response format. Same standard as the credit risk and loan portfolio projects. |
| **Docker** | Containerized deployment, consistent with every other project. |

---

## Data Flow

**Current implementation — honest, static starting point:**

    Kaggle RBA dataset (33M rows, CSV)
        ↓
    Polars reads directly from file
        ↓
    Feature engineering (geo-jump, device-change, login velocity)
        ↓
    Train/test split → baseline model (Random Forest / XGBoost)
        ↓
    MLflow tracks the experiment
        ↓
    SHAP explains each prediction
        ↓
    FastAPI serves the trained model
        ↓
    PostgreSQL logs every live prediction (audit trail)

**Target production architecture — what this would look like deployed:**

    Real login events, continuous
        ↓
    Application servers emit events → Kafka (same pattern as fraud detection)
        ↓
    Consumer reads from Kafka, computes features in real time
    against each user's last known state, held in Redis (feature store)
        ↓
    Trained model scores the session in real time
        ↓
    Score + SHAP explanation → SIEM / case management (same pipeline as fraud)
        ↓
    PostgreSQL logs the prediction
        ↓
    Airflow periodically retrains on newly confirmed labels (weekly)

This project is being built in two deliberate sequences — Sequence 1 (Phases 1-5: data, features, model, explainability, calibration, serving) builds and proves the core model correctly in isolation. Sequence 2 (Phase 6: Kafka + Redis) adds the real-time simulation layer on top of a model already trusted to be correct, not built simultaneously, so that if something breaks, it's clear whether the problem is the model or the streaming infrastructure. There's no fixed timeline here — the goal is genuine engineering depth, not speed.

## Project Structure

    account-takeover-detection/
    ├── data/                        # session/event datasets (gitignored)
    ├── models/                      # trained model artifacts (.pkl, gitignored)
    ├── config/
    │   └── config.yaml              # thresholds: impossible-travel speed, alert cutoff
    ├── notebooks/
    │   ├── 01_EDA_and_Feature_Engineering.ipynb
    │   └── 02_Modeling_Baseline_vs_Sequence.ipynb
    ├── src/
    │   ├── app.py                   # FastAPI serving endpoint
    │   ├── feature_engineering.py   # velocity, device-change, geo-jump functions
    │   ├── model_calibration.py     # Platt Scaling / Isotonic Regression
    │   └── shap_explain.py          # SHAP explanation generation
    ├── scripts/
    │   ├── download_data.py         # pulls the RBA dataset via kagglehub
    │   ├── train_baseline.py        # runs training — Random Forest / XGBoost
    │   ├── train_sequence.py        # runs training — LSTM (stretch goal)
    │   ├── kafka_producer.py        # Phase 6 — simulates live login stream
    │   └── kafka_consumer.py        # Phase 6 — real-time scoring from Kafka
    ├── dags/
    │   └── retrain_dag.py           # Airflow — weekly retrain on confirmed labels
    ├── vertex/
    │   └── training_job_config.py   # Vertex AI Training Job submission
    ├── tests/
    │   ├── test_feature_engineering.py
    │   └── test_api.py
    ├── grafana/
    │   └── dashboard.json           # dashboard definition
    ├── mlruns/                      # MLflow experiment tracking (gitignored)
    ├── outputs/                     # proof-of-work screenshots and results
    ├── .env                         # DB credentials, API keys (gitignored)
    ├── .gitignore
    ├── Dockerfile
    ├── docker-compose.yml
    ├── prometheus.yml               # Prometheus scrape config
    ├── requirements.txt
    └── README.md

---

*Part of a 30-week MLOps programme · Tunajijengea wenyewe 🇰🇪*
