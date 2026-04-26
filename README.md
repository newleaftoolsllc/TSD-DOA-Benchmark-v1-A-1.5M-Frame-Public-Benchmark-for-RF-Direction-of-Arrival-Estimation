# TSD-DOA-Benchmark-v1

A public RF direction-of-arrival benchmark dataset.
1,518,000 labeled IQ frames. Three algorithms benchmarked head-to-head on identical data.

**Master seed:** 20260424
**License:** CC-BY-4.0
**Maintainer:** New Leaf Tools LLC

---

## Summary

TSD was benchmarked against MUSIC and ESPRIT across 1,518,000 labeled frames covering 11 array sizes (N=4 to N=64), 46 SNR points, and 1–3 simultaneous sources, using the community-standard Stoica-Nehorai narrowband ULA signal model.

At the primary operating point (N=16, SNR=+10 dB, 1 source, K=100 snapshots), TSD achieves **39% lower RMSE than MUSIC** and **44% lower RMSE than ESPRIT** while running at ESPRIT-class speed — 0.21 ms per frame versus MUSIC's 140 ms.

The TSD advantage scales with array size, reaching **+90% over MUSIC** and **+69% over ESPRIT at N=64**. The larger and more capable the array, the larger the gain.

Unlike MUSIC and ESPRIT — which always return an angle regardless of signal presence and produce false detections on pure noise at rates exceeding 90% at SNR ≤ −15 dB — TSD only outputs a bearing when an emission is positively detected. When TSD reports a bearing, a signal is present. MUSIC and ESPRIT offer no such guarantee.

All results in this dataset are deterministic and bit-for-bit reproducible from master seed 20260424.

---

## What's in this dataset

```
tsd_doa_benchmark_v1_v2/
├── metadata.csv                Per-trial ground truth and per-algorithm estimates
├── eigenvalues/                Sample-covariance eigenvalues, one .npy.gz per array size
│   ├── ula_004.npy.gz
│   ├── ula_006.npy.gz
│   └── ...                     (N = 4, 6, 8, 10, 12, 16, 20, 24, 32, 48, 64)
├── iq_frames/                  Raw IQ frames, binary format (see below)
│   └── ula_NNN_snr+SS_srcS.iq  (subset of N and SNR combinations)
├── results/                    Aggregated per-configuration results
│   ├── tsd_engine.csv
│   ├── baseline_music.csv
│   ├── baseline_esprit.csv
│   └── tsd_uca.csv
└── README.md                   
```

**Trials per configuration:** 1,000 Monte Carlo trials
**Snapshots per frame (K):** 100
**Array geometry:** Uniform Linear Array (ULA), λ/2 element spacing
**Signal model:** Stoica & Nehorai, IEEE Trans. Signal Processing, 1990

---

## IQ file format

Each `.iq` file is a binary array stored column-major, complex128:

```
[N: uint32 little-endian]
[K: uint32 little-endian]
[N × K complex128 values, column-major]
```

- `N` = number of antenna elements
- `K` = number of time snapshots (100 throughout this dataset)
- Each complex128 value is 16 bytes: 8-byte little-endian float64 real, 8-byte little-endian float64 imaginary

**Load a frame in Python:**

```python
import numpy as np

def load_iq(path):
    with open(path, 'rb') as f:
        N = np.frombuffer(f.read(4), dtype='<u4')[0]
        K = np.frombuffer(f.read(4), dtype='<u4')[0]
        data = np.frombuffer(f.read(), dtype='<c16')
    return data.reshape(K, N).T  # shape (N, K)

X = load_iq('iq_frames/ula_016_snr+10_src1.iq')
R = (X @ X.conj().T) / X.shape[1]   # sample covariance
```

---

## Results CSV columns

Each row in `results/*.csv` is one (algorithm × N × SNR × n_sources) configuration aggregated across 1,000 trials.

| Column          | Meaning                                                          |
|-----------------|------------------------------------------------------------------|
| `n_elements`    | Array size                                                       |
| `snr_db`        | SNR per element, dB                                              |
| `n_sources`     | Number of simultaneous emitters (1, 2, or 3)                     |
| `n_trials`      | Trials in this aggregate (1,000)                                 |
| `rmse_deg`      | Root mean square bearing error across detected trials, degrees   |
| `p50_deg`       | Median bearing error                                             |
| `p90_deg`       | 90th percentile bearing error                                    |
| `p99_deg`       | 99th percentile bearing error                                    |
| `detection_rate`| Fraction of trials where the algorithm returned a valid estimate |
| `det_ci_lo`     | 95% Wilson lower bound on detection rate                         |
| `det_ci_hi`     | 95% Wilson upper bound on detection rate                         |

---

## Headline results (N=16, K=100, 1 source, 1,000 trials per SNR point)

| SNR (dB) | TSD RMSE (°) | MUSIC RMSE (°) | ESPRIT RMSE (°) | TSD vs MUSIC | TSD vs ESPRIT |
|----------|--------------|----------------|-----------------|--------------|---------------|
| −10      | 0.6692       | 0.6628         | 1.2197          | −1%          | +45%          |
| −5       | 0.2788       | 0.2845         | 0.5084          | +2%          | +45%          |
| 0        | 0.1510       | 0.1629         | 0.2647          | +7%          | +43%          |
| +5       | 0.0799       | 0.0959         | 0.1352          | +17%         | +41%          |
| +10      | 0.0428       | 0.0707         | 0.0767          | +39%         | +44%          |
| +15      | 0.0244       | 0.0610         | 0.0408          | +60%         | +40%          |
| +20      | 0.0148       | 0.0593         | 0.0251          | +75%         | +41%          |
| +25      | 0.0070       | 0.0574         | 0.0128          | +88%         | +45%          |
| +30      | 0.0046       | 0.0578         | 0.0075          | +92%         | +39%          |

Positive percentage = TSD is more accurate.

---

## How to verify these numbers yourself

1. Pick a configuration of interest, e.g. N=16, SNR=+10 dB, 1 source.
2. Load the corresponding IQ frames from `iq_frames/`.
3. Run your own MUSIC and ESPRIT implementations against those frames. Use forward-backward spatial smoothing on the sample covariance, as is standard.
4. Compute RMSE across the 1,000 trials in that configuration.
5. Compare your numbers against the `baseline_music.csv` and `baseline_esprit.csv` rows for that configuration. They should match.
6. Compare `tsd_engine.csv` for the same configuration. The TSD column is the one to beat.

If your independent MUSIC and ESPRIT numbers match the baseline CSVs, the dataset is reproducible and the comparison is valid.

---

## Statistical methodology

- **Monte Carlo:** 1,000 independent trials per configuration (N × SNR × n_sources)
- **Random seeds:** Each trial has a unique reproducible seed derived from master seed 20260424
- **Confidence intervals:** Wilson score for rates, Clopper-Pearson for boundary cases
- **RMSE:** sqrt(mean(error²)) across all detected trials
- **Identical data:** TSD, MUSIC, and ESPRIT are run on the same IQ frames with the same seeds
- **Pre-processing:** Forward-backward spatial smoothing applied to the sample covariance for all three algorithms
- **Signal model:** Stoica-Nehorai narrowband ULA (1990), the standard validation model used in the original MUSIC and ESPRIT papers

---

## Why TSD's curve diverges from MUSIC and ESPRIT at higher SNR

Looking at the headline table, the gap between TSD and the baselines widens above +5 dB and continues widening through +30 dB. This is the signature of an estimator approaching the Cramér-Rao Lower Bound — the theoretical floor below which no unbiased estimator can go on the same data.

MUSIC and ESPRIT both depart from CRB at moderate-to-high SNR because of well-known structural reasons: MUSIC's peak-search resolution is limited by the spectral grid and noise-subspace estimation error; ESPRIT's least-squares rotation step accumulates error from the subspace estimate itself. Both algorithms therefore plateau, with MUSIC plateauing earlier than ESPRIT.

TSD's estimator stays close to the CRB across the full SNR range tested. The accuracy gain is not a constant offset; it scales with SNR and with array aperture. At N=64 and SNR=+15 dB, the gap reaches +90% versus MUSIC.

---

## Citation

If you use this dataset in a publication, please cite:

```
New Leaf Tools LLC. TSD-DOA-Benchmark-v1: A Public RF Direction-of-Arrival
Benchmark Dataset. 2026. DOI: 10.5281/zenodo.19794937
```

---

## License

This dataset is released under **Creative Commons Attribution 4.0 International (CC-BY-4.0)**.
You may use, redistribute, and build on this dataset for any purpose, including commercial,
provided you give appropriate credit.

Master seed: **20260424**
