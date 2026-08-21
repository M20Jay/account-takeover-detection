# Account Takeover Detection — Behavioral Anomaly Classifier

**Author:** Martin James Ng'ang'a | Nairobi, Kenya 🇰🇪
**Status:** ⏳ Planned — Week 13
**Stack:** Sequence modeling (LSTM / gradient-boosted feature engineering) · Vertex AI Training · FastAPI · PostgreSQL · Langfuse · Docker · GCP

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

---

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

## Project Structure

    account-takeover-detection/
    ├── data/                        # session/event datasets (gitignored)
    ├── notebooks/
    │   ├── 01_EDA_and_Feature_Engineering.ipynb
    │   └── 02_Modeling_Baseline_vs_Sequence.ipynb
    ├── src/
    │   ├── app.py                   # FastAPI serving endpoint
    │   ├── feature_engineering.py   # velocity, device-change, geo-jump features
    │   ├── train_baseline.py        # Random Forest / XGBoost baseline
    │   └── train_sequence.py        # LSTM sequence model (stretch goal)
    ├── vertex/
    │   └── training_job_config.py   # Vertex AI Training Job submission
    ├── outputs/                     # proof-of-work screenshots and results
    ├── docker-compose.yml
    ├── requirements.txt
    └── README.md

---

*Part of a 30-week MLOps programme · Tunajijengea wenyewe 🇰🇪*
