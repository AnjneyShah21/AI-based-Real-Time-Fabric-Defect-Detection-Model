# Research Paper Issues — LoomVisionAI

> **Date**: 22 June 2026  
> **Status**: Post Embroidery-suppression fix (`411e81b`)  
> **Scope**: Every technical claim in the IEEE Access paper cross-referenced against source code  

---

## ❌ Factual Errors (Wrong Values)

These are cases where a specific number/value in the paper does not match the actual code.

### 1. SSIM Tile Size — Paper says `61 × 61`, code says `64 × 64`

| | Paper | Code |
|---|---|---|
| **Value** | 61 × 61 tiles | 64 × 64 pixels |
| **Location (Paper)** | Section III.C — Pattern Anomaly Detection |
| **Location (Code)** | `src/detection.py` Line 238: `tile_size: int = 64` |

**Fix**: Change "61 × 61" to "64 × 64" in the paper.

---

### 2. Stain Detection Threshold — Paper says `35`, code says `50`

| | Paper | Code |
|---|---|---|
| **Value** | 35 | 50 |
| **Location (Paper)** | Section III.D — Heuristic Defect Classification |
| **Location (Code)** | `src/heuristic_classifier.py` Line 63: `if color_diff > 50` |
| **Note** | Code comment says: *"raised from 35 to avoid false stain labels"* — paper references the old value |

**Fix**: Change "35" to "50" in the paper.

---

### 3. Embroidery Color Std Threshold — Paper says `> 15`, code says `> 30`

| | Paper | Code |
|---|---|---|
| **Value** | > 15 | > 30 |
| **Location (Paper)** | Section III.D — Heuristic Defect Classification |
| **Location (Code)** | `src/heuristic_classifier.py` Line 77: `np.mean(std_color) > 30` |

**Fix**: Change "15" to "30" in the paper.

---

### 4. Embroidery Edge Density Threshold — Paper says `> 0.14`, code says `> 0.25`

| | Paper | Code |
|---|---|---|
| **Value** | > 0.14 | > 0.25 |
| **Location (Paper)** | Section III.D — Heuristic Defect Classification |
| **Location (Code)** | `src/heuristic_classifier.py` Line 77: `edge_density > 0.25` |

**Fix**: Change "0.14" to "0.25" in the paper.

---

### 5. Crease Rejection — Aspect Ratio — Paper says `≤ 5`, code says `≤ 8.0`

| | Paper | Code |
|---|---|---|
| **Value** | ≤ 5 | ≤ 8.0 |
| **Location (Paper)** | Section III.C — Structural Defect Detection (Crease Rejection) |
| **Location (Code)** | `src/detection.py` Line 101: `if aspect > 8.0` |

**Fix**: Change "5" to "8.0" in the paper.

---

### 6. Crease Rejection — Extent — Paper says `≥ 0.25`, code says `≥ 0.15`

| | Paper | Code |
|---|---|---|
| **Value** | ≥ 0.25 | ≥ 0.15 |
| **Location (Paper)** | Section III.C — Structural Defect Detection (Crease Rejection) |
| **Location (Code)** | `src/detection.py` Line 107: `if extent < 0.15` |

**Fix**: Change "0.25" to "0.15" in the paper.

---

## ❌ Behavioral Errors (Wrong Algorithm / Logic Described)

These are cases where the paper describes a mechanism that works fundamentally differently from what the code actually does.

### 7. Design Suggestion Engine — Paper says "K-Means Clustering", code uses Counter-based histogram

| | Paper | Code |
|---|---|---|
| **Algorithm** | K-Means clustering in the HSV colour space | Pixel-by-pixel HSV-to-name mapping + `Counter.most_common()` |
| **Location (Paper)** | Section VI.B — Design Suggestion Engine |
| **Location (Code)** | `src/design_suggestions.py` Lines 142–167 |
| **Details** | The code maps each pixel to a colour name via hardcoded HSV thresholds (`_hsv_to_colour_name()`), then counts occurrences with `Counter(names).most_common(k)`. No K-Means or any clustering algorithm exists in the codebase. |

**Fix**: Replace "K-Means clustering" with "HSV-threshold pixel classification with frequency counting" in the paper. Alternatively, implement actual K-Means using `sklearn.cluster.KMeans` or `cv2.kmeans`.

---

### 8. Color Reference Algorithm — Paper says "EMA", code uses Cumulative Running Average

| | Paper | Code |
|---|---|---|
| **Algorithm** | Exponentially weighted average (EMA) | Cumulative average with decreasing alpha = 1/n |
| **Location (Paper)** | Section III.C.2 — Color Anomaly Detection |
| **Location (Code)** | `src/detection.py` Lines 153–154 |
| **Details** | An EMA uses a **fixed** alpha (like the pattern detector's α = 0.05). The colour reference uses `alpha = 1.0 / (self._ref_count + 1)` which **decreases** with every frame (1.0, 0.5, 0.33, 0.25...). This produces a cumulative running average, NOT an EMA. Newer frames get progressively LESS weight — the opposite of what "exponentially weighted" implies. |

**Fix**: Either (a) change the paper to say "running cumulative average" instead of "exponentially weighted average", or (b) change the code to use a fixed alpha (e.g., `alpha = 0.1`) to make it a true EMA.

---

### 9. Temporal Debouncing — Paper says enforced at 3 frames, code only tracks (never enforces)

| | Paper | Code |
|---|---|---|
| **Claim** | "Requires a defect to persist for at least three consecutive frames" | Counter increments/resets but is NEVER used as a gate condition |
| **Location (Paper)** | Section III.F — Debouncing |
| **Location (Code)** | `src/prediction_engine.py` Lines 273–279 |
| **Details** | The code increments `self.consecutive_defect_frames` when a defect is detected and resets it to 0 otherwise. However, there is **no** `if consecutive_defect_frames >= 3` check anywhere — the `has_defect` flag is returned directly without any debouncing gate. The comment on Line 275 even says "require 2 consecutive frames" (not 3 as the paper says), but even this is not enforced. |

**Fix**: Either (a) implement actual debouncing logic by adding a gate:
```python
if has_defect:
    self.consecutive_defect_frames += 1
    if self.consecutive_defect_frames < 3:
        has_defect = False  # suppress until confirmed
else:
    self.consecutive_defect_frames = 0
```
Or (b) remove the debouncing claim from the paper.

---

### 10. LSTM Warmup — Paper says "50 frames", should say "50 clean (non-defect) frames"

| | Paper | Code |
|---|---|---|
| **Claim** | "After a warmup period of 50 frames" | 50 entries in `loss_history`, which only grows during **non-defect** frames |
| **Location (Paper)** | Section III.E — Temporal Sequence Model |
| **Location (Code)** | `src/sequence_model.py` Lines 75–87 |
| **Details** | The `loss_history.append()` is inside an `if not is_spatial_defect:` block (Line 75). If the system sees defects during warmup, it takes **more** than 50 total frames to warm up — only clean frames count. This is a meaningful distinction that the paper omits. |

**Fix**: Change "warmup period of 50 frames" to "warmup period of 50 non-defective frames" in the paper.

---

## ⚠️ Internal Contradictions

### 11. Debouncing — Claimed as feature in Section III, listed as limitation in Section VII

| Section | What it says |
|---|---|
| **Section III.F** | "The final stage prevents false alerts by requiring a defect to persist for at least three consecutive frames." |
| **Section VII.B (Limitations)** | Lists the lack of temporal consistency as a limitation. |
| **Reality** | The code tracks consecutive frames but never enforces debouncing. Section VII is more honest. |

**Fix**: Resolve the contradiction. Either implement debouncing (fixing Section III) or remove the claim from Section III (keeping Section VII's limitation statement).

---

## ⚠️ Minor Issues / Imprecise Statements

### 12. WhatsApp Alert — Paper says "Inspection instructions" are included

| | Paper | Code |
|---|---|---|
| **Claim** | "Includes: Defect category, Detection timestamp, Inspection instructions" |
| **Reality** | Message says: "Please check the loom immediately" — not really "inspection instructions" |
| **Location (Code)** | `src/notifications.py` Lines 26–30 |

**Fix**: Either expand the WhatsApp message to include actual inspection instructions (e.g., "Check warp thread tension at section X"), or change "Inspection instructions" to "Immediate alert message" in the paper.

---

### 13. Debouncing comment says "2 frames" but paper says "3 frames"

| | Paper | Code Comment |
|---|---|---|
| **Value** | 3 consecutive frames | "require 2 consecutive frames to confirm" (Line 275) |
| **Note** | Neither is actually enforced — purely cosmetic inconsistency |

**Fix**: Align the comment with whichever value is chosen when implementing the actual debouncing.

---

## ✅ Issues Already Fixed

### ~~Embroidery Suppression~~

| | Before | After (commit `411e81b`) |
|---|---|---|
| **Issue** | Paper said both Embroidery + Wrinkle are suppressed; code only suppressed Wrinkle | Code now suppresses both |
| **File** | `src/prediction_engine.py` Line 267 |
| **Status** | ✅ **FIXED** |

---

## Summary

| Category | Count | IDs |
|---|---|---|
| ❌ Factual Errors (wrong numbers) | 6 | #1–#6 |
| ❌ Behavioral Errors (wrong algorithm/logic) | 4 | #7–#10 |
| ⚠️ Internal Contradictions | 1 | #11 |
| ⚠️ Minor Issues | 2 | #12–#13 |
| ✅ Already Fixed | 1 | Embroidery Suppression |
| **Total remaining issues** | **13** | |
