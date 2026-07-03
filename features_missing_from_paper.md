# Features Present in Code but Missing from Research Paper

> **Date**: 22 June 2026  
> **Purpose**: Lists every notable feature implemented in the LoomVisionAI codebase that is either completely absent or severely underrepresented in the IEEE Access research paper. Adding these to the paper would strengthen the novelty claims and improve publishability.

---

## 1. Motion-Adaptive Threshold Scaling

**Where in Code**: `src/prediction_engine.py` Lines 131–160

**What it does**: The PatchCore anomaly threshold is dynamically scaled based on the belt's motion state using a dual-mode strategy:

| Belt State | Scale Factor | Effect |
|---|---|---|
| Continuous motion (>30 frames) | **0.7×** | Increases sensitivity — PatchCore has adapted to motion patterns, so lower the bar to catch subtle defects |
| Transitioning / settling | **1.5×** | Decreases sensitivity — motion blur inflates scores, so raise the bar to prevent false positives |

**Why it matters**: No existing PatchCore paper modulates the anomaly threshold based on conveyor belt state. This is a novel domain-specific contribution that directly addresses the challenge of industrial deployment on a moving belt.

**Suggested paper section**: Add a subsection under Section III titled _"Motion-Aware Threshold Modulation"_ with the formula: `τ_motion = τ_adaptive × α_state`

---

## 2. Selective Memory Bank Update Policy

**Where in Code**: `src/prediction_engine.py` Lines 82–85, `src/dynamic_patchcore.py` Lines 242–248

**What it does**: The memory bank follows a contamination-resistant update policy:

| Condition | Memory Bank Updated? | Reason |
|---|---|---|
| Belt **stopped** + frame is **clean** | ✅ Yes | Ideal learning conditions — sharp, defect-free fabric |
| Belt **moving continuously** (>30 frames) + clean | ✅ Yes | Motion is now "normal" — safe to learn motion patterns |
| Belt **transitioning** (settling / starting) | ❌ No | Blur/vibration artifacts would contaminate the bank |
| Frame has a **detected defect** | ❌ No | Defective patterns must not enter the normal reference |

The warmup counter (`_frame_count`) also only increments during stopped-belt frames, meaning the memory bank is built exclusively from sharp, stationary fabric.

**Why it matters**: Standard PatchCore uses a fixed, pre-computed memory bank. This online, contamination-aware update policy is a form of curriculum learning for the memory bank — a publishable contribution that makes the system robust to real-world deployment noise.

**Suggested paper section**: A formal Algorithm box (Algorithm 1) with pseudocode for the update policy.

---

## 3. Conveyor Belt State Machine — Dual-Signal Fusion with Settling Phase

**Where in Code**: `src/conveyor_state.py` (entire file, 176 lines)

**What the paper says**: Brief mention of belt state detection.

**What's not described**: The full engineering detail of the state machine:

1. **Dual-signal fusion**: A frame is considered "in motion" only when **both** signals agree:
   - Laplacian variance < blur threshold (motion blur detected)  
   - Frame difference > motion threshold (actual pixel movement)
   - Using AND (not OR) prevents false triggers from low-texture fabrics that inherently have low Laplacian variance

2. **Majority voting**: Motion decisions are smoothed over a 5-frame history window — belt is "moving" only if the majority of recent frames agree

3. **Three-state machine**: `moving` → `settling` → `stopped`
   - `settling` phase (0.3s) lets mechanical vibrations die out before declaring the belt stopped
   - This prevents the AI from processing shaky, half-blurred frames

4. **Software-defined replacement**: Replaces hardware sensors (Arduino / photoelectric) with pure computer vision — zero additional hardware cost

**Why it matters**: This is a practical contribution for Industry 4.0 applications. Most papers require hardware sensors; this eliminates that dependency entirely.

**Suggested paper section**: Elevate to a full subsection with a state diagram and algorithm pseudocode.

---

## 4. Dual Preprocessing Pipeline (LAB + HSV CLAHE)

**Where in Code**: `src/preprocessing.py` (65 lines — two functions)

**What it does**: The system uses **two independent preprocessing paths**, each optimized for a different detection task:

### Path 1: `apply_preprocessing()` — For Structural/Pattern Detection
```
BGR → Gaussian Blur (3×3) → LAB → Extract L-channel → CLAHE (clipLimit=2.0) → Gaussian Blur (9×9)
```
- L-channel carries luminance only — **illumination-invariant**, works regardless of fabric colour
- Used by `DefectDetector.detect_structural_anomaly()` for Canny edge detection

### Path 2: `apply_color_preprocessing()` — For Colour Anomaly Detection
```
BGR → Gaussian Blur (3×3) → HSV → CLAHE on V-channel → Merge back → BGR
```
- Preserves Hue and Saturation (critical for stain/dye detection)
- Normalizes only Value (brightness) to compensate for uneven lighting
- Used by `DefectDetector.detect_color_anomaly()`

**Why it matters**: Most fabric defect detection papers use a single preprocessing path. The dual-path design is a deliberate engineering choice that ensures structural detection is colour-blind while colour detection preserves chromatic information. This prevents the common failure mode where CLAHE on RGB/BGR distorts hue values.

---

## 5. Latest-Frame-Only WebSocket Streaming Architecture

**Where in Code**: `flask_server.py` Lines 390–410

**What it does**: When a mobile phone streams camera frames to the backend via WebSocket, the server uses a **latest-frame-only** strategy:

```python
# Just store the latest frame. The background thread will pick it up.
# This prevents WebSocket queue buildup and camera freezing.
MOBILE_LATEST_FRAME = frame
```

Instead of queuing frames (which would build up when ML processing is slower than camera capture), the server overwrites a single variable. The background processing thread always picks up the **most recent** frame and discards any intermediate ones.

**Why it matters**: This is a producer-consumer pattern with latest-only semantics, specifically designed to prevent:
- WebSocket message queue buildup (which causes memory growth)
- Browser camera freezing (which happens when the WebSocket back-pressures the client)
- Processing stale frames (the ML always sees the newest data)

This is a practical systems engineering insight relevant to real-time ML deployment.

---

## 6. Multi-Scale Feature Fusion Details (Layer 2 + Layer 3)

**Where in Code**: `src/dynamic_patchcore.py` Lines 145–182

**What the paper says**: Mentions ResNet-18 and Layer 2 + Layer 3 hooks, but doesn't explain the fusion process.

**What's not described**:
1. **Layer 2**: Produces `[1, 128, 28×28]` feature maps capturing fine thread-level textures
2. **Layer 3**: Produces `[1, 256, 14×14]` feature maps capturing broader structural patterns
3. **Spatial alignment**: Layer 2's 28×28 maps are downsampled to 14×14 using `F.adaptive_avg_pool2d` so both layers share the same spatial grid
4. **Concatenation**: Features are concatenated along the channel dimension → `[1, 384, 14, 14]`
5. **Reshape**: Flattened to `[196, 384]` — 196 spatial patches, each with a 384-dimensional feature vector
6. **L2 normalization**: Each patch vector is L2-normalized for cosine similarity computation

**Why it matters**: The specific fusion strategy (which layers, how they're aligned, the resulting dimensionality) is reproducibility-critical information. Reviewers need this to replicate the system.

---

## 7. Camera Auto-Detection with Platform-Aware Scoring

**Where in Code**: `src/camera.py` Lines 75–129

**What it does**: On macOS, the camera controller:
1. Uses `system_profiler SPCameraDataType` to enumerate connected cameras
2. Assigns a **score** to each camera:
   - +1 for each external keyword match (e.g., "android", "samsung", "logitech", "pixel")
   - -1 for each built-in keyword match (e.g., "facetime", "macbook", "isight")
3. Selects the highest-scoring camera automatically

On Linux/Windows, falls back to probing OpenCV indices 0–2.

Additionally supports:
- **IP Webcam mode**: Connects to a phone's IP webcam via URL (from `.env` file)
- **Camera switching**: Cycles through available cameras at runtime
- **Warm-up retry**: Tries up to 50 reads (2.5 seconds) to handle slow USB camera initialization

**Why it matters**: Demonstrates real-world deployment engineering — the system works out-of-the-box when a user plugs in any phone or webcam, without manual camera index configuration.

---

## 8. Asynchronous WhatsApp Alerts with Dual Rate Limiting

**Where in Code**: `src/notifications.py` + `src/database.py` Lines 72–76

**What the paper says**: Mentions WhatsApp alerts via Twilio with a 10-second cooldown.

**What's not described**:
- Alerts are sent **asynchronously** via `threading.Thread(target=send_whatsapp_alert, ..., daemon=True)` — the ML pipeline never blocks waiting for Twilio's API response
- **Graceful degradation**: If Twilio credentials are not configured (`"your_" in account_sid`), the system silently skips notifications instead of crashing
- Alerts are triggered from `DatabaseLogger.log_defect()`, not from the prediction engine — separating detection from notification concerns

**Why it matters**: Shows production-grade engineering: non-blocking I/O, graceful fallback, and separation of concerns.

---

## 9. SQLite Database Schema with Session Tracking

**Where in Code**: `src/database.py` Lines 24–53

**What the paper says**: Mentions SQLite for defect logging.

**What's not described**: The actual database schema:
```sql
CREATE TABLE defects (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT NOT NULL,
    defect_type TEXT NOT NULL,
    image_path TEXT NOT NULL,
    confidence REAL DEFAULT 0.0,
    anomaly_score REAL DEFAULT 0.0,
    engine_used TEXT DEFAULT 'unknown',
    session_id TEXT DEFAULT 'default'
)
```

Notable details:
- Stores **which engine** detected the defect (`patchcore`, `opencv_structural`, `opencv_color`, `opencv_pattern`, `temporal`)
- Supports **session tracking** via `session_id` — enables comparing defect rates across inspection sessions
- Includes **schema migration** logic (`ALTER TABLE ... ADD COLUMN`) for backward compatibility with older databases
- Defect images are saved to disk and referenced by path — not stored as BLOBs

---

## 10. Periodic Debug Health Logging

**Where in Code**: `src/prediction_engine.py` Lines 105–109

**What it does**: Every 30th frame, the prediction engine prints a health-check line:
```
[PredictionEngine] Frame #91 | belt=stopped | PatchCore warmed=True (10/10) | memory=10 patches | thr=0.612
```

This provides at-a-glance visibility into:
- Frame counter
- Belt state
- PatchCore warmup status (current/required frames)
- Memory bank size
- Current adaptive threshold

**Why it matters**: Essential for debugging and monitoring in real-world deployment. Shows the system is designed for production observability, not just research prototyping.

---

## 11. Evaluation Framework with Confusion Matrix

**Where in Code**: `evaluate_classifier.py` (104 lines)

**What the paper says**: Mentions evaluation methodology but doesn't present the actual evaluation infrastructure.

**What's not described**:
- A complete, runnable evaluation script that:
  1. Initializes the full `PredictionEngine`
  2. Loads images from `data/validation/{normal, broken_thread, loose_weave, stain}/`
  3. Processes each image through the complete AI pipeline
  4. Computes per-class precision, recall, F1-score
  5. Generates a full confusion matrix
  6. Saves results to `output/reports/evaluation_results.json`

**Why it matters**: This script IS the reproducibility mechanism. It should be referenced in the paper's evaluation section with instructions on how to run it.

---

## 12. `is_learning_safe` Guard — Conditional AI Learning

**Where in Code**: `src/prediction_engine.py` Lines 82–85

```python
is_continuous = self._continuous_motion_frames > 30
is_learning_safe = is_stopped or is_continuous
```

**What it does**: Defines a boolean flag `is_learning_safe` that gates when the PatchCore memory bank is allowed to update. Learning is "safe" only when:
- The belt is stopped (sharp frames), OR
- The belt has been moving continuously for 30+ frames (motion is now the steady state)

During transitions (first 30 frames of motion, or settling), learning is blocked.

**Why it matters**: This is the mechanism that prevents memory bank contamination during transient states. It's the control logic behind Feature #2 (Selective Memory Bank Update) and deserves explicit documentation.

---

## Summary — Priority for Paper

| # | Feature | Novelty Impact | Effort to Add |
|---|---------|---------------|--------------|
| 1 | Motion-Adaptive Threshold Scaling | ⭐⭐⭐ Very High | Easy (write-up only) |
| 2 | Selective Memory Bank Update Policy | ⭐⭐⭐ Very High | Easy (write-up + pseudocode) |
| 3 | Belt State Machine — Dual-Signal Fusion | ⭐⭐ High | Easy (write-up + state diagram) |
| 4 | Dual Preprocessing Pipeline | ⭐⭐ High | Easy (write-up only) |
| 5 | Latest-Frame WebSocket Architecture | ⭐ Medium | Easy (1 paragraph) |
| 6 | Multi-Scale Feature Fusion Details | ⭐⭐ High | Easy (expand existing section) |
| 7 | Camera Auto-Detection | ⭐ Low | Easy (1 paragraph) |
| 8 | Async WhatsApp with Dual Rate Limiting | ⭐ Low | Easy (expand existing section) |
| 9 | SQLite Schema + Session Tracking | ⭐ Low | Easy (add schema figure) |
| 10 | Periodic Debug Health Logging | ⭐ Low | Trivial (1 sentence) |
| 11 | Evaluation Framework | ⭐⭐ High | Easy (reference the script) |
| 12 | `is_learning_safe` Guard | ⭐⭐ High | Covered by #2 |
