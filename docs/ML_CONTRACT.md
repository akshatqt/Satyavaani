# SatyaVaani ML Contract

This document defines the shared technical contract for the SatyaVaani ML team.

All ML contributors must follow these definitions and values unless the team explicitly agrees to change them.

This document is the source of truth for the shared ML interface between:

- Aryan — Data Preparation
- Akshat — Core Model Training
- Sanjeevani — Fusion and Risk Scoring

---

# 1. ML Team Responsibilities

## Aryan — Data Preparation

Responsible for:

- Dataset collection and organization
- Data cleaning
- Metadata structuring
- Train/validation/test splitting
- Shared audio preprocessing
- Audio windowing
- Data augmentation
- Producing model-ready training data

Aryan's output is the clean, structured and model-ready data consumed by the model-training pipeline.

Aryan is not responsible for training the primary detection models.

---

## Akshat — Core Model Training

Responsible for:

- Spectral/deepfake detection
- Prosody detection
- Speaker identity detection
- Model training
- Model tuning
- Model evaluation
- Model artifacts
- Model inference interfaces

The primary model candidates are:

- AASIST-L for spectral/deepfake detection
- A suitable ML model for prosody analysis
- ECAPA-TDNN for speaker identity/voice consistency

The exact model architecture may be changed if experiments demonstrate a better or more practical approach.

---

## Sanjeevani — Fusion and Risk Scoring

Responsible for:

- Combining detector outputs
- LightGBM fusion model
- Risk-score generation
- Risk-level classification
- Limited fusion evaluation

Sanjeevani is not responsible for training the individual detection models.

---

# 2. Shared Project Constants

The following values are shared across the entire ML pipeline.

```python
PROJECT_NAME = "SatyaVaani"

TARGET_SAMPLE_RATE = 16000

WINDOW_SIZE_SEC = 2.0
HOP_SIZE_SEC = 0.5

WINDOW_SIZE_SAMPLES = 32000
HOP_SIZE_SAMPLES = 8000

RISK_SCORE_MIN = 0
RISK_SCORE_MAX = 100

SAFE_THRESHOLD = 40
HIGH_RISK_THRESHOLD = 75

These values must not be independently changed by individual contributors.

If a change is required, the ML team must agree on the change and update this document.

3. Audio Contract

All standard ML audio input must use:

Sample rate: 16000 Hz
Channels: mono
Window size: 2.0 seconds
Hop size: 0.5 seconds

Therefore:

WINDOW_SIZE_SAMPLES = 32000
HOP_SIZE_SAMPLES = 8000

The streaming/backend team must provide audio in this format to the ML inference layer.

If incoming audio is not in this format, the system must normalize it before model inference.

4. Labels

The canonical dataset labels are:

LABEL_REAL = "REAL"
LABEL_FAKE = "FAKE"

These labels should be used consistently throughout the ML pipeline.

If an external dataset uses different labels, the preprocessing pipeline must convert them to the canonical labels.

For example, dataset-specific labels such as:

bonafide
spoof
genuine
synthetic
real
fake

must be mapped to the project's canonical representation.

5. Dataset Split Contract

The ML pipeline must maintain separate:

Training set
Validation set
Test set

The test set must not be used for model tuning.

Where the dataset permits it, speaker-aware splitting should be preferred to reduce data leakage.

The exact split strategy must be documented by the data-preparation pipeline.

6. Audio Windowing

The standard windowing configuration is:

WINDOW_SIZE_SEC = 2.0
HOP_SIZE_SEC = 0.5

At 16 kHz:

WINDOW_SIZE_SAMPLES = 32000
HOP_SIZE_SAMPLES = 8000

Conceptually:

Window 1: 0.0s - 2.0s
Window 2: 0.5s - 2.5s
Window 3: 1.0s - 3.0s
Window 4: 1.5s - 3.5s
...

This means:

Each ML window contains 2 seconds of audio.
A new window begins every 0.5 seconds.
Consecutive windows overlap by 1.5 seconds.

This configuration must remain consistent between training and inference.

7. Data Preprocessing

The shared preprocessing pipeline should normalize audio to the project's standard format.

The expected basic pipeline is:

Raw Audio
    ↓
Audio Loading
    ↓
Channel Normalization
    ↓
Sample Rate Normalization
    ↓
Basic Audio Validation
    ↓
2-second Windowing
    ↓
Model-specific Processing

The shared preprocessing pipeline is owned by Aryan.

Individual models may require additional model-specific preprocessing.

Any model-specific preprocessing must be documented and must not silently contradict the shared audio contract.

8. Data Augmentation

Training-time augmentation may be used to improve robustness to realistic communication conditions.

Potential augmentations include:

Background noise
Codec degradation
Packet loss
Jitter
Reverberation
Volume variation
Other realistic communication-channel distortions

Augmentation should have a clear purpose.

Augmentation must not unnecessarily alter the semantic label.

For example:

REAL audio + background noise
→ REAL

and:

FAKE audio + codec degradation
→ FAKE

unless a specific transformation is intentionally being used to model a different problem.

Augmentation is primarily a training-time operation and should not automatically be applied to validation/test data.

9. Detector Architecture

SatyaVaani uses three complementary detection signals.

                    AUDIO
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      SPECTRAL      PROSODY     IDENTITY
          |           |           |
          v           v           v
       AASIST-L     PROSODY      ECAPA
          |          MODEL       TDNN
          |           |           |
          v           v           v
    artifact_score  prosody_score  speaker_mismatch_score
          |           |           |
          +-----------+-----------+
                      |
                      v
                   FUSION
                      |
                      v
                 risk_score
                      |
                      v
                 risk_level

The three detectors are complementary.

10. Spectral / Deepfake Detector

The primary candidate is:

AASIST-L

The purpose of this detector is to identify acoustic/spectral characteristics associated with synthetic or manipulated speech.

Its standardized output is:

artifact_score: float

The score must be normalized to:

0.0 - 1.0

Meaning:

0.0 → low suspicion of synthetic/manipulated audio
1.0 → high suspicion of synthetic/manipulated audio

Higher values always mean greater suspicion.

If another model is eventually selected instead of AASIST-L, it must still produce the same standardized output:

artifact_score
11. Prosody Detector

The prosody detector analyzes characteristics of how speech is produced.

Potential characteristics include:

Pitch
Pitch variation
Speaking rate
Rhythm
Pauses
Timing
Intonation
Other relevant temporal speech characteristics

The exact feature set and model architecture will be determined during experimentation.

Its standardized output is:

prosody_score: float

The score must be normalized to:

0.0 - 1.0

Meaning:

0.0 → low suspicion
1.0 → high suspicion

Higher values always mean greater suspicion.

12. Speaker Identity Detector

The primary candidate is:

ECAPA-TDNN

The purpose of this component is to determine whether the live voice is consistent with the expected/reference speaker.

Conceptually:

Reference Speech
      ↓
ECAPA-TDNN
      ↓
Reference Speaker Representation


Live Speech
      ↓
ECAPA-TDNN
      ↓
Live Speaker Representation


Reference Representation
          +
Live Representation
          ↓
Similarity / Comparison
          ↓
speaker_mismatch_score

Its standardized output is:

speaker_mismatch_score: float

The score must be normalized to:

0.0 - 1.0

Meaning:

0.0 → strong speaker consistency
1.0 → strong speaker mismatch

Higher values always mean greater suspicion.

13. Score Direction

All three detector scores must use the same direction.

0.0 = less suspicious
1.0 = more suspicious

Therefore:

higher score = higher suspicion

This applies to:

artifact_score
prosody_score
speaker_mismatch_score

This rule is critical because these values are later used by the fusion model.

A detector must not return a score where higher means "more genuine" under one of these standardized field names.

14. Standard Detector Output

Every detector must eventually expose its standardized result using the agreed field name.

{
    "artifact_score": float,
    "prosody_score": float,
    "speaker_mismatch_score": float
}

A detector that is not applicable to a particular component should not fabricate a value.

The integration design will define how unavailable signals are handled.

15. Fusion Input

The fusion layer initially receives:

artifact_score
prosody_score
speaker_mismatch_score

All three values are:

float
0.0 - 1.0

The initial fusion feature set is therefore:

artifact_score
prosody_score
speaker_mismatch_score

Additional features may be introduced later only if:

Reliable data exists for them.
The feature has a clear purpose.
The ML team agrees to add it.
The ML contract is updated.
16. Fusion Model

The primary fusion model is:

LightGBM

The fusion model learns how to combine the three detector signals into one overall risk score.

Conceptually:

artifact_score
       |
prosody_score
       |-----> LightGBM -----> risk_score
       |
speaker_mismatch_score

The exact LightGBM hyperparameters are not fixed by this document and may be tuned during experimentation.

17. Fusion Baseline

The project should maintain a simple deterministic fusion baseline.

This ensures that the complete system can operate even if the LightGBM fusion model is not yet finished.

A weighted combination may be used as the initial baseline.

Example concept:

artifact_score
prosody_score
speaker_mismatch_score
        ↓
weighted combination
        ↓
0.0 - 1.0
        ↓
0 - 100

The exact baseline weights must be documented when implemented.

The baseline is not necessarily the final production fusion method.

18. Risk Score

The fusion layer produces:

risk_score: float

The allowed range is:

0 - 100

The meaning is:

0   → very low overall suspicion
100 → very high overall suspicion

The standardized constants are:

RISK_SCORE_MIN = 0
RISK_SCORE_MAX = 100

If the fusion model naturally produces a probability in the range 0.0 - 1.0, it may be converted to the project risk-score range.

The direction must remain:

higher risk_score = greater suspicion
19. Risk Level

The standardized risk levels are:

SAFE
SUSPECT
HIGH_RISK

The thresholds are:

SAFE_THRESHOLD = 40
HIGH_RISK_THRESHOLD = 75

The classification logic is:

if risk_score < SAFE_THRESHOLD:
    risk_level = "SAFE"
elif risk_score < HIGH_RISK_THRESHOLD:
    risk_level = "SUSPECT"
else:
    risk_level = "HIGH_RISK"

Therefore:

0–39   → SAFE
40–74  → SUSPECT
75–100 → HIGH_RISK

These thresholds are shared across the ML, backend and frontend components.

20. Final ML Result Contract

The final ML result must use these exact field names:

{
    "artifact_score": float,
    "prosody_score": float,
    "speaker_mismatch_score": float,
    "risk_score": float,
    "risk_level": str
}

Example:

{
    "artifact_score": 0.82,
    "prosody_score": 0.61,
    "speaker_mismatch_score": 0.74,
    "risk_score": 83.0,
    "risk_level": "HIGH_RISK"
}

These field names must remain consistent across:

ML code
Fusion code
Backend
API
WebSocket messages
Frontend
Tests
Documentation
21. Audio Window Metadata Contract

Every processed audio window should be associated with:

call_id
window_id
timestamp_start
timestamp_end
sample_rate

Example:

{
    "call_id": "CALL_001",
    "window_id": 12,
    "timestamp_start": 6.0,
    "timestamp_end": 8.0,
    "sample_rate": 16000
}
call_id

Identifies the call.

The same call_id must remain associated with all windows from that call.

window_id

Identifies the individual window within a call.

For example:

0
1
2
3
...
timestamp_start

Start time of the audio window in seconds.

timestamp_end

End time of the audio window in seconds.

22. Model Training and Evaluation

The core model-training pipeline should maintain separate training, validation and test data.

Useful evaluation metrics may include:

Precision
Recall
F1-score
ROC-AUC
Equal Error Rate (EER)
False Positive Rate
False Negative Rate
Confusion Matrix

The exact metrics used may differ depending on the model.

Training accuracy alone must not be used as the primary indication of model quality.

23. Generalization

The ML system should be evaluated for robustness to conditions that differ from the training data.

Where data permits, evaluation should consider:

Unseen speakers
Different recording conditions
Background noise
Codec degradation
Different spoofing/deepfake methods
Different languages or accents where relevant

The goal is to reduce overfitting to a particular dataset or attack generator.

24. Data Leakage Prevention

The ML pipeline must avoid data leakage.

Examples of problematic leakage include:

The same recording appearing in both training and test sets.
Highly duplicated recordings appearing across splits.
Speaker overlap that makes the evaluation artificially easy when speaker separation is required.
Test-set information being used to tune the final model.

The data-preparation pipeline must document the split strategy.

25. Training vs Inference

Training-time operations and inference-time operations must remain clearly separated.

For example:

Training:
Raw audio
→ preprocessing
→ augmentation
→ model training

whereas:

Inference:
Live audio
→ preprocessing
→ model inference

Training-only augmentation must not automatically be applied to live inference.

The shared audio assumptions must remain consistent.

26. Model Artifacts

Large model artifacts should not be committed directly to the normal Git repository.

Examples include:

*.pt
*.pth
*.ckpt
*.onnx
*.safetensors
*.joblib
*.pkl

The project will later decide how trained models are stored and distributed.

Possible options include:

Git LFS
External model storage
Cloud/object storage
Model registry

The choice will be documented separately.

27. Dependencies Between ML Team Members

The intended workflow is:

ARYAN
Data Preparation
      ↓
Model-ready data
      ↓
AKSHAT
Core Model Training
      ↓
Three detector outputs
      ↓
SANJEEVANI
Fusion / LightGBM
      ↓
Final risk score

Aryan's work should provide the data required by Akshat.

Akshat's work should provide the detector outputs required by Sanjeevani.

Sanjeevani's final risk output should be usable by the backend and frontend.

28. Compatibility Rules

All ML contributors must use the shared field names and conventions defined in this document.

Do not independently create alternative names such as:

score
fake_score
spoof_score
genuine_probability
identity_score
final_score

when a standardized field already exists.

Use:

artifact_score
prosody_score
speaker_mismatch_score
risk_score
risk_level

If a genuinely new field is required, discuss it with the team before introducing it.

29. Configuration

Shared constants should eventually be stored in a common configuration location rather than being manually duplicated across multiple files.

The implementation should avoid hardcoding the same value in many different places.

For example, do not write:

sample_rate = 16000

in ten separate files.

Instead, the project should eventually have a shared configuration source from which the components obtain:

TARGET_SAMPLE_RATE
WINDOW_SIZE_SEC
HOP_SIZE_SEC
WINDOW_SIZE_SAMPLES
HOP_SIZE_SAMPLES
SAFE_THRESHOLD
HIGH_RISK_THRESHOLD

The exact configuration implementation will be decided when the repository structure is finalized.

30. Changes to This Contract

This document is the source of truth for the shared ML interface.

If implementation code and this document disagree, the discrepancy must be resolved before integration.

Any change to:

Shared constants
Score names
Score direction
Risk thresholds
Audio format
Standard output fields
Label conventions

must be communicated to all ML contributors.

The updated contract must be committed to the repository before dependent code is changed.

31. Current Scope

This contract intentionally defines the shared interfaces and conventions.

It does NOT permanently define:

Exact AASIST-L architecture/configuration
Exact prosody model architecture
Exact ECAPA-TDNN configuration
Exact feature-extraction implementation
Exact LightGBM hyperparameters
Exact training hyperparameters
Exact model-export format
Exact model-serving implementation
Exact deployment architecture

Those are implementation decisions and may be changed based on experiments, performance, available hardware and hackathon constraints.

Any decision that affects the shared interface must be reflected back into this contract.

32. Quick Reference
Audio
Sample Rate: 16000 Hz
Channels: Mono
Window: 2.0 seconds
Hop: 0.5 seconds
Detector Scores
artifact_score:          0.0 - 1.0
prosody_score:           0.0 - 1.0
speaker_mismatch_score:  0.0 - 1.0

For all three:

0.0 = less suspicious
1.0 = more suspicious
Final Risk
risk_score: 0 - 100
Risk Levels
0–39   → SAFE
40–74  → SUSPECT
75–100 → HIGH_RISK
Shared Thresholds
SAFE_THRESHOLD = 40
HIGH_RISK_THRESHOLD = 75
Final Output
{
    "artifact_score": float,
    "prosody_score": float,
    "speaker_mismatch_score": float,
    "risk_score": float,
    "risk_level": str
}