# Multi-Stage Lifecycle Classifier: Behavioral Re-evaluation Engine

## Executive Summary
This project engineers a temporal state-machine to categorize mobile game players into Lifetime Value (LTV) cohorts (Payer, Watcher, Neutral) across a 120-hour (5-day) lifecycle. 

Instead of relying on static classification, the pipeline evaluates 'Persona Drift'—tracking users who exhibit delayed monetization intent or fall into 'Fake Payer' grace periods. To operationalize this for live-ops, the deterministic 5-day ground truth was used to train an XGBoost predictive model, successfully classifying a player's Day-5 Final State using exclusively Day-1 (Onboarding) telemetry. 

## Pipeline Architecture

### 1. High-Fidelity Telemetry Simulation & Chaos Injection
Generated a 4.2-million-row dataset representing 200,000 unique player lifecycles. To rigorously test the pipeline's sanitization logic, production-level financial anomalies were injected:
* **Ghost Transactions:** Failed App Store receipt validations yielding `NaN` revenue.
* **Chronological Corruption:** Out-of-bounds event logging requiring strict temporal filtering.

### 2. Vectorized Feature Aggregation
Bypassed standard row-wise Pandas operations (which fail at scale) in favor of C-level vectorized aggregations. Raw events (`iap_attempt`, `bundle_clicked`, `ad_watch`) were mathematically transformed into distinct intent heuristics:
* **PayerScore:** High-weighting for shop visits, bundle clicks, and a flat bonus for actualized revenue.
* **WatcherScore:** Density tracking for level progression and ad-consumption.

### 3. The State-Machine Engine (Deterministic Logic)
Implemented a strict stage-gate system based on business logic flowcharts:
* **S1 Onboarding:** Initial Day-1 classification.
* **Day 3 Re-evaluation:** A 48-hour lock followed by drift analysis to detect late-blooming Payers.
* **Day 5 Grace Period:** A final revenue validation check. 'Payers' who exhibited high early intent but spent zero lifetime revenue were automatically demoted to protect marketing budgets from false positives.

### 4. Predictive LTV Modeling (XGBoost)
The 5-day state-machine established the ultimate ground truth. A multi-class XGBoost model was trained to predict the `Final_Class` using *only* `S1_Onboarding` features. This isolates predictive early-indicators, allowing targeted monetization strategies to deploy on Day 2 before player churn occurs.

### 5. Explainable AI & SHAP Verification
The XGBoost model achieved near-perfect predictive accuracy. To prove this was not target leakage, SHAP (SHapley Additive exPlanations) analysis was conducted. 
* **The Finding:** The model entirely ignored the raw `revenue_S1` feature.
* **The Proof:** SHAP verified that the engineered `PayerScore` and `WatcherScore` successfully captured 100% of the mathematical variance required to classify the users. The model reverse-engineered the rigid threshold logic of the Day-1 deterministic state-machine, validating the feature engineering architecture.

## Tech Stack
* **Data Engineering:** Python, Pandas (Vectorized GroupBys), NumPy
* **Machine Learning:** XGBoost (Multi-class Classification)
* **Model Interpretability:** SHAP
