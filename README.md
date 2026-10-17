# IntelliGuard
**A secure, explainable, privacy-preserving insider-threat detection system.**
Taneesha Sahay (K048) · Gauri Ghadage (K014)

IntelliGuard learns each user's normal behaviour from activity logs, flags unusual days (for example
night-time bulk copying), explains *why*, and is built so the tool itself is hard to attack:
validated input, encrypted storage, strong authentication, role-based access, and a tamper-evident audit trail.

> **Status: ~50% implementation milestone.** The core pipeline, security layer and ML are implemented and tested.
> The web UI, real-dataset evaluation (CERT/BETH) and deployment are still to do. See [Status](#status).

## Architecture (matches the block and class diagrams)

| Block | Class(es) | File |
|---|---|---|
| Data sources | `generate_logs`, `load_cert_r42` | `intelliguard/data/` |
| **M1** Secure log ingestion | `LogIngestor` | `ingestion.py` |
| **M2** Preprocessing (20 features, 10×20 windows) | `Preprocessor` | `preprocessing.py` |
| **M3** Encrypted storage (AES-256-GCM) | `EncryptedStorage`, `DataCipher` | `storage.py`, `security/crypto.py` |
| **M4** Anomaly detection | `AnomalyModel`, `StackedLSTM`, `IsolationForestModel` | `models/` |
| M4 federated learning + LDP | `ClientNode`, `FederatedServer`, `LDPMechanism` | `federated/` |
| **M5** Risk scoring and alerts | `RiskScorer`, `Alert` | `scoring.py` |
| **XAI** engine | `Explainer`, `ShapExplainer`, `LimeExplainer` | `explain.py` |
| **M6** RBAC dashboard (logic) | `Dashboard`, `AccessControl` | `dashboard.py`, `security/rbac.py` |
| **M7** Auth and sessions | `AuthService`, `User`, `Totp` | `security/auth.py` |
| **M8** Tamper-proof audit log | `AuditLog`, `AuditEntry` | `audit.py` |

## Quick start

**Google Colab (easiest):** open `notebooks/IntelliGuard_Colab.ipynb` → *Runtime → Run all* (about 5–8 min on CPU).
The notebook writes the package to disk and runs the full demo.

**Local:**
```bash
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                       # then fill in the two secrets
python -m pytest -q                                        # 48 tests
PYTHONPATH=. python scripts/run_demo.py                    # quick smoke run (Isolation Forest only)
python scripts/build_notebook.py                           # rebuild the Colab notebook from the sources
```

## Security features (and how they are tested)
- **Injection defence:** allow-list schema, flat scalars only, rejects `$`-operators and SQL patterns; parameterized SQL everywhere.
- **Encryption at rest:** AES-256-GCM, fresh nonce per record; tampering is detected (`InvalidTag`). Keys come from the environment, never from source.
- **Authentication:** Argon2id passwords (12-character minimum), TOTP MFA (checked against the RFC 6238 test vector), 15-minute JWTs, account lockout, uniform error messages.
- **Authorization:** deny-by-default RBAC; every denial and admin action is written to the audit chain.
- **Audit integrity:** SHA-256 hash chain. Detects edits, deletions and reordering. Truncation or a fully rewritten chain is detected only if you keep the latest hash somewhere the admin cannot write (`AuditLog.head()`).
- **Privacy:** federated learning keeps raw logs on each client; clients add Laplace noise to their updates (LDP).

## Results: read before quoting numbers
Evaluated on **synthetic** CERT-style logs (40 users, 90 days, 6 insiders). The attacks are fairly separable, so
scores near 1.0 are **not** a forecast for real data. Run the notebook for the exact table; in our runs the LSTM
detected most insiders (recall ≈ 0.96) versus ≈ 0.49 for Isolation Forest at a similar false-alert rate, and LDP
only degraded detection at very small ε (≈ 0.1). Results vary slightly between runs (TensorFlow is not fully deterministic).

## Status
| Done | Not done / limitations |
|---|---|
| M1–M5, M8, XAI, dashboard *logic*, auth, RBAC, federated + LDP | Web UI (dashboard is a Python class, no front-end) |
| 48 automated tests, Colab demo | HTTP/API layer, cookie handling, TLS deployment (only a TLS 1.3 context helper exists) |
| CERT r4.2 loader written | CERT loader untested on the full dataset; **BETH loader not started** |
| | LDP ε is per coordinate: no formal end-to-end privacy guarantee |
| | No validated defence against data poisoning beyond input validation (trimmed retraining is experimental) |
| | XAI explains 20 window-summary features, not individual events. LIME is a local linear approximation: it ranks features like SHAP but its magnitudes can differ (about 2x in our runs) |
| | Key management is env-var only; audit anchoring is manual; no Docker/CI |

## Datasets
- **CERT Insider Threat** (CMU): https://kilthub.cmu.edu/articles/dataset/Insider_Threat_Test_Dataset/12841247 → `data/cert_loader.py`
- **BETH** (syscall logs): planned.

## Publishing to GitHub
1. Create an **empty** repository on github.com (no README/licence).
2. `bash scripts/push_to_github.sh https://github.com/<your-username>/intelliguard.git`
