**Full Technical Paper**
Asymmetric Extrapolation in SQL Server Sampled Statistics
https://github.com/clunn7363-dev/sql-server-sampled-statistics-asymmetry/blob/main/whitepaper.md

# Asymmetric Extrapolation in SQL Server Sampled Statistics

This repository accompanies a technical whitepaper describing an asymmetry in SQL Server sampled statistics generation, where deterministic extrapolation is applied to point frequency estimates (`EQ_ROWS`) but not to sampled distinct counts used for range estimation (`DISTINCT_RANGE_ROWS` and `AVG_RANGE_ROWS`).

On high‑cardinality data with bounded per‑value frequency, this asymmetry can cause average range estimates to inflate systematically as effective sampling rates fall, despite unchanged true data distribution. The behaviour is observable, independent of skew, and reproducible on real production systems.

The paper presents empirical evidence, explains the underlying estimator mechanics, and outlines low‑cost engine‑level approaches to restore internal consistency using signals SQL Server already computes.

This material is intended as technical feedback and design discussion, not as a diagnostic or remediation guide.
