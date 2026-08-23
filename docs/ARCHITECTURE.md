# SatyaVaani System Architecture

This document defines the shared system architecture for the SatyaVaani project.

It is the source of truth for how the major components interact.

The architecture is designed to allow the six team members to work independently while maintaining clear interfaces between their components.

---

# 1. Project Overview

SatyaVaani is a real-time voice impersonation and deepfake detection system.

The system is designed to process live or simulated call audio, divide the audio into short windows, analyze those windows using multiple ML signals, combine the signals into an overall risk score, and present the result to the user.

The high-level flow is:

```text
Audio Source
    ↓
Real-Time Audio Gateway
    ↓
Audio Windows
    ↓
ML Inference Service
    ↓
Multiple ML Detectors
    ↓
Fusion / Risk Scoring
    ↓
Risk Result
    ↓
Frontend
2. Team Responsibilities
2.1 Aryan — Data Preparation

Aryan owns the ML data pipeline.

Responsibilities:

Dataset collection
Dataset organization
Data cleaning
Metadata structuring
Train/validation/test splitting
Audio preprocessing
Audio windowing for training data
Data augmentation
Producing model-ready data

Aryan's output is consumed by the ML training pipeline.

2.2 Akshat — Core ML and Inference

Akshat owns the primary ML intelligence.

Responsibilities:

Spectral/deepfake detection
Prosody detection
Speaker identity detection
Model training
Model evaluation
Model inference
Combining the trained detector outputs with the fusion layer
Providing the ML inference interface

The primary model candidates are:

AASIST-L
A suitable prosody model
ECAPA-TDNN

The exact models may change based on experiments and implementation feasibility.

2.3 Sanjeevani — Fusion and Risk Scoring

Sanjeevani owns the final ML-level fusion component.

Responsibilities:

Receiving detector scores
LightGBM fusion
Risk-score generation
Risk-level classification
Limited fusion evaluation

Sanjeevani does not train the primary detection models.

2.4 Ayush — Real-Time Audio Gateway

Ayush owns the real-time systems side.

Responsibilities:

Live audio ingestion
Audio buffering
Sliding-window creation
Go media gateway
Rust components where required
gRPC communication with the ML service
Call/session management
Streaming reliability
Low-latency communication
Handling dropped connections and incomplete audio

Ayush does not own the ML models.

2.5 Manav — Cloud, Containerization and Infrastructure

Manav owns the infrastructure layer.

Responsibilities:

Docker
Docker Compose
Service networking
Environment configuration
Deployment
ML container requirements
CPU/GPU configuration
Health checks
Logging
Resource monitoring
Scaling strategy
CI/CD if time permits
Production-style deployment considerations

Manav should prioritize a working reproducible prototype over unnecessary infrastructure complexity.

2.6 Yoshita — UI/UX

Yoshita owns the frontend experience.

Responsibilities:

UI/UX design
Frontend implementation
Risk visualization
Call/session information
Displaying ML results
User-facing alerts
Frontend integration with the backend/gateway

The frontend should consume standardized backend results rather than implementing ML logic itself.

3. High-Level Architecture

The intended architecture is:

                         SATYAVAANI
                              |
                              |
                       Audio Source
                              |
                 +------------+------------+
                 |                         |
                 v                         v
          Real Call Source          Demo/Simulated
          SIP/RTP/WebRTC              Audio Source
                 |                         |
                 +------------+------------+
                              |
                              v
                    +-------------------+
                    |   Ayush Gateway   |
                    |     Go / Rust     |
                    +---------+---------+
                              |
                              |
                             gRPC
                              |
                              v
                    +-------------------+
                    |   ML Inference    |
                    |     Service       |
                    +---------+---------+
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
           Spectral        Prosody       Identity
            Detector        Detector      Detector
             AASIST-L        Model         ECAPA
                |             |             |
                v             v             v
          artifact_score  prosody_score  speaker_mismatch_score
                |             |             |
                +-------------+-------------+
                              |
                              v
                    +-------------------+
                    |   Fusion Layer    |
                    |     LightGBM       |
                    +---------+---------+
                              |
                              v
                         risk_score
                              |
                              v
                         risk_level
                              |
                              v
                    +-------------------+
                    | Backend / Gateway |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |     Frontend      |
                    |      Yoshita      |
                    +-------------------+

                 Manav provides the
            containerization/deployment layer
                 around the services.
4. Core Data Flow

The complete system follows this conceptual flow:

1. Audio enters the system.

2. Ayush's gateway receives the audio.

3. The gateway buffers incoming audio.

4. Audio is divided into 2-second windows.

5. Consecutive windows begin every 0.5 seconds.

6. Each window is sent to the ML inference service.

7. The ML service performs the required preprocessing.

8. The spectral detector produces artifact_score.

9. The prosody detector produces prosody_score.

10. The identity detector produces speaker_mismatch_score.

11. The three scores are passed to the fusion layer.

12. The fusion layer produces risk_score.

13. risk_score is converted into risk_level.

14. The result is returned to the gateway/backend.

15. The frontend receives the result.

16. The frontend displays the current call risk.
5. Audio Windowing

The standard audio configuration is defined in:

docs/ML_CONTRACT.md

The shared values are:

TARGET_SAMPLE_RATE = 16000

WINDOW_SIZE_SEC = 2.0
HOP_SIZE_SEC = 0.5

WINDOW_SIZE_SAMPLES = 32000
HOP_SIZE_SAMPLES = 8000

Conceptually:

Incoming Audio:

0.0 ───────── 2.0
     Window 1

0.5 ───────── 2.5
     Window 2

1.0 ───────── 3.0
     Window 3

1.5 ───────── 3.5
     Window 4

The same windowing assumptions must be used consistently between training and inference.

6. Real-Time Gateway

Ayush's gateway is responsible for connecting the audio source to the ML service.

The gateway should:

Receive audio
Buffer audio
Maintain call/session state
Generate windows
Assign call IDs
Assign window IDs
Send windows to ML
Receive ML results
Forward results to the appropriate backend/frontend layer

The gateway should not contain the ML models.

7. Development / Demo Mode

The architecture must support a simulated audio source.

This is important because real SIP/RTP/WebRTC integration may take significant time.

The demo pipeline can therefore be:

Audio File / Simulated Stream
        ↓
Ayush Gateway
        ↓
2-second Windows
        ↓
ML Inference Service
        ↓
Risk Result
        ↓
Frontend

This allows the team to demonstrate the complete system without requiring a fully operational real-world telephony integration.

A simulated audio source should use the same ML interface as the real streaming source.

8. Real Call Mode

If real communication integration is implemented, the expected flow is:

Phone / Communication System
        ↓
SIP / RTP / WebRTC
        ↓
Ayush Gateway
        ↓
ML Inference Service

The exact telephony implementation is not fixed by this document.

The system should remain modular so that the audio source can be replaced without changing the ML model interface.

9. ML Inference Service

The ML inference service is responsible for running trained models against incoming audio windows.

It should:

Receive an audio window.
Validate the request.
Apply required inference preprocessing.
Run the spectral detector.
Run the prosody detector.
Run the identity detector where reference information is available.
Obtain the standardized detector scores.
Run the fusion layer.
Generate the final risk result.
Return the standardized response.

The service should not require the gateway to know the internal model architecture.

10. ML Detector Interface

The three ML detector outputs are:

artifact_score
prosody_score
speaker_mismatch_score

Each score must be:

0.0 - 1.0

For all three:

0.0 = less suspicious
1.0 = more suspicious

The exact models producing these values may change without changing the external interface.

11. ML Fusion Interface

The fusion layer receives:

artifact_score
prosody_score
speaker_mismatch_score

The primary fusion model is:

LightGBM

It produces:

risk_score
risk_level

The risk score is:

0 - 100

The risk levels are:

0–39   → SAFE
40–74  → SUSPECT
75–100 → HIGH_RISK

The exact fusion implementation should remain internal to the ML service.

12. Standard ML Request

The semantic structure of a request to the ML service should contain:

{
  "call_id": "CALL_001",
  "window_id": 12,
  "timestamp_start": 6.0,
  "timestamp_end": 8.0,
  "sample_rate": 16000,
  "audio": "<audio window>"
}

The exact serialization and transport format may differ depending on the final gRPC implementation.

The semantic fields must remain consistent.

13. Standard ML Response

The ML service should return:

{
  "call_id": "CALL_001",
  "window_id": 12,
  "risk_score": 83.0,
  "risk_level": "HIGH_RISK",
  "artifact_score": 0.82,
  "prosody_score": 0.61,
  "speaker_mismatch_score": 0.74
}

The exact protocol representation may differ.

The field names must remain consistent.

14. Call and Window Identification

Each call receives a unique:

call_id

Each audio window within that call receives:

window_id

Example:

CALL_001
    |
    +-- window 0
    +-- window 1
    +-- window 2
    +-- window 3
    +-- ...

This allows results to be associated with the correct call and audio window.

15. Backend / Gateway Boundary

The gateway/backend layer is responsible for communication with the ML service.

It should not duplicate ML logic.

For example, the gateway should not independently calculate:

artifact_score
prosody_score
speaker_mismatch_score
risk_score

Those values belong to the ML service.

The gateway transports and manages the results.

16. Frontend Boundary

The frontend consumes standardized results.

The frontend may display:

Current risk level
Risk score
Detector scores
Call ID
Current window
Timeline/history
Alerts
Other approved UI information

The frontend must not contain the ML model or duplicate risk-classification logic.

The frontend should use the standardized risk values returned by the backend.

17. Infrastructure Layer

Manav's infrastructure layer surrounds the application services.

The initial service layout is conceptually:

                    Docker Environment
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     ML Service       Gateway          Frontend

Supporting services may be introduced later if necessary.

Potential technologies include:

Docker
Docker Compose
Redis
Kafka
Kubernetes
ONNX Runtime
TensorRT

These are optional technologies.

They should only be introduced when they provide a clear benefit.

18. Docker

Each major service should be independently runnable.

Initial conceptual services:

satya-ml
satya-gateway
satya-frontend

Potential supporting services:

satya-redis
satya-kafka

Redis and Kafka should not be introduced unless required.

The target development experience is:

Clone repository
        ↓
Configure environment
        ↓
docker compose up
        ↓
Complete system starts
19. Environment Configuration

Configuration should be externalized.

Potential configuration variables include:

ML_SERVICE_HOST
ML_SERVICE_PORT

GATEWAY_HOST
GATEWAY_PORT

MODEL_PATH

LOG_LEVEL

Optional variables may include:

REDIS_URL
KAFKA_BROKERS

Only variables actually required by the implementation should be added.

Secrets must never be committed to GitHub.

20. Model Paths

ML model files must not use developer-specific paths.

Do not use:

/Users/Akshat/Desktop/model.pth

Instead, use configuration:

MODEL_PATH=/models/model.pth

The exact environment variable and model directory structure will be finalized during implementation.

21. Health Checks

Each major service should expose a health mechanism.

Example:

GET /health

Possible response:

{
  "status": "ok"
}

The exact protocol may differ for non-HTTP services.

Health checks should allow the infrastructure layer to determine whether a service is alive.

22. Service Readiness

A service being alive does not necessarily mean it is ready.

For example:

Container starts
      ↓
Python runtime starts
      ↓
ML dependencies initialize
      ↓
Model weights load
      ↓
Inference runtime initializes
      ↓
ML service becomes READY

The system should distinguish between:

SERVICE_ALIVE

and:

SERVICE_READY

where practical.

23. CPU / GPU

The ML service should support the hardware configuration that is actually available.

Possible inference environments include:

CPU
GPU
Edge hardware

The project must measure actual model requirements before deciding that a GPU is mandatory.

If GPU deployment is required, the infrastructure documentation should specify:

GPU requirement
CUDA/runtime requirement
Expected memory requirement
Deployment limitations
24. ONNX Runtime / TensorRT

ONNX Runtime and TensorRT are optimization/deployment options.

They are not mandatory components.

The preferred order is:

1. Working Python inference
2. Measure inference latency
3. Identify bottlenecks
4. Investigate optimization
5. Use ONNX/TensorRT if beneficial

The team should not sacrifice the working prototype merely to introduce an optimization technology.

25. Redis

Redis is optional.

Potential uses include:

Temporary call state
Latest risk result
Session information
Temporary metadata

Redis should only be introduced if the application actually requires shared fast state.

For a simple prototype, in-memory state may be sufficient.

26. Kafka

Kafka is optional.

It may become useful for:

Event processing
High-volume streaming
Decoupled services
Future scalability

However, Kafka introduces additional infrastructure complexity.

For the initial prototype, direct gRPC communication should be preferred when it is sufficient.

27. Kubernetes

Kubernetes is optional.

The infrastructure priority is:

1. Docker
2. Docker Compose / simple deployment
3. Service networking
4. Health checks
5. Logging
6. Resource monitoring
7. Kubernetes if genuinely useful and time permits

A working prototype has higher priority than unnecessary orchestration complexity.

28. Logging

Services should produce useful operational logs.

Examples:

CALL_STARTED
WINDOW_RECEIVED
ML_REQUEST_SENT
ML_RESPONSE_RECEIVED
HIGH_RISK_DETECTED
CALL_ENDED
SERVICE_ERROR

Logs must not unnecessarily contain sensitive raw audio.

The logging format should be consistent enough to allow debugging across services.

29. Privacy

SatyaVaani processes potentially sensitive voice information.

The preferred architecture is:

Audio
  ↓
Inference
  ↓
Result
  ↓
Raw audio discarded

rather than permanent storage of raw recordings.

Raw audio must not be permanently stored unless explicitly required by the implementation and approved by the team.

30. Secrets

The repository must never contain:

API keys
Passwords
Private certificates
Cloud credentials
Database passwords
Other sensitive secrets

Use environment configuration or an appropriate secret-management system.

The .env file must remain uncommitted.

An .env.example file may be committed if required.

31. Service Ownership
Component	Owner	Primary Technology
Data pipeline	Aryan	Python
ML models	Akshat	Python / ML frameworks
ML fusion	Sanjeevani	Python / LightGBM
Real-time gateway	Ayush	Go / Rust
Infrastructure	Manav	Docker / Cloud
Frontend	Yoshita	Frontend stack

Ownership does not mean that contributors cannot help each other.

It means each component has a clear primary owner.

32. Repository Ownership

The GitHub repository is owned and maintained by Akshat.

All team members contribute through Git.

Team members should avoid directly modifying another person's component unless:

They are collaborating with the owner.
The owner has requested the change.
The change is agreed upon by the team.
33. Interface Ownership

The main interfaces are:

Ayush Gateway
      ↓
ML Inference Service

and:

ML Inference Service
      ↓
Backend / Gateway
      ↓
Frontend

The interfaces should remain stable even if internal implementations change.

34. ML ↔ Gateway Contract

The gateway provides:

call_id
window_id
timestamp_start
timestamp_end
sample_rate
audio window

The ML service returns:

call_id
window_id
artifact_score
prosody_score
speaker_mismatch_score
risk_score
risk_level

The gateway should not need to know the internal ML architecture.

35. Gateway ↔ Frontend Contract

The frontend should receive enough information to display the current call state.

Potential fields include:

call_id
window_id
risk_score
risk_level
artifact_score
prosody_score
speaker_mismatch_score
timestamp

The exact frontend API will be finalized during implementation.

36. Error Handling

Each service should handle failures gracefully.

Possible failures include:

Invalid audio
Incomplete audio window
ML service unavailable
gRPC timeout
Network failure
Model loading failure
Invalid ML response
Call disconnection
Container failure

The system should avoid crashing the entire application because one request failed.

Errors should be logged appropriately.

37. Latency

The system is intended to operate as a real-time detection system.

The target is to provide a useful risk result shortly after enough audio has been collected.

The project should measure:

Audio capture delay
+
Windowing delay
+
Network delay
+
ML inference latency
+
Fusion latency
+
Frontend update latency

The actual achievable latency must be measured rather than assumed.

38. Scalability

The initial implementation should prioritize correctness and simplicity.

Future scaling may involve:

Multiple gateway instances
        ↓
Load balancing
        ↓
Multiple ML inference workers
        ↓
Shared state / event infrastructure

Technologies such as Redis, Kafka and Kubernetes may be introduced later if required.

They are not mandatory for the initial prototype.

39. Development Strategy

The project should be developed in stages.

Stage 1 — ML Prototype
Audio file
    ↓
ML preprocessing
    ↓
Models
    ↓
Fusion
    ↓
Risk result
Stage 2 — ML Service
Audio request
    ↓
ML inference service
    ↓
Risk result
Stage 3 — Gateway Integration
Simulated stream
    ↓
Go gateway
    ↓
ML service
    ↓
Risk result
Stage 4 — Frontend Integration
Gateway / Backend
    ↓
Frontend
    ↓
Live risk visualization
Stage 5 — Containerization
Docker
    ↓
Multiple services
    ↓
Docker Compose
    ↓
Complete reproducible system
Stage 6 — Real Communication Integration

If feasible:

Real communication source
    ↓
SIP / RTP / WebRTC
    ↓
Gateway
    ↓
ML service
    ↓
Frontend
40. Demo Fallback Strategy

The hackathon prototype must have a fallback path.

If real-time communication integration is incomplete, the team should still be able to demonstrate:

Pre-recorded audio
       ↓
Simulated streaming
       ↓
2-second windows
       ↓
ML inference
       ↓
Fusion
       ↓
Frontend

This ensures that an incomplete external integration does not prevent demonstration of the core AI system.

41. What Is Mandatory

The minimum viable architecture requires:

Audio Source
     ↓
Gateway
     ↓
ML Service
     ↓
Detection Models
     ↓
Fusion
     ↓
Risk Result
     ↓
Frontend

The following are not mandatory for the initial prototype:

Kafka
Redis
Kubernetes
TensorRT
ONNX Runtime
Real SIP integration
Production-scale cloud infrastructure

These may be added if time and requirements justify them.

42. Source of Truth

The following documents define the project's shared architecture and contracts:

docs/ARCHITECTURE.md
docs/ML_CONTRACT.md

ML_CONTRACT.md defines the ML-specific interfaces and shared ML values.

ARCHITECTURE.md defines how the complete system fits together.

If implementation code conflicts with these documents, the conflict must be resolved before integration continues.

43. Architecture Change Policy

The architecture may evolve during development.

However, changes that affect another team member must be communicated before implementation.

Examples:

Changing gRPC to another protocol
Changing audio window size
Changing ML output field names
Changing risk thresholds
Changing gateway responsibilities
Adding Redis or Kafka
Changing frontend/backend interfaces
Changing model-serving architecture

The relevant documentation must be updated when an architectural decision changes.

44. Final System View

The complete intended system can be summarized as:

                         SATYAVAANI
                              |
                              v
                    AUDIO SOURCE / CALL
                              |
                              v
                    +-------------------+
                    |   AYUSH GATEWAY   |
                    |     Go / Rust     |
                    +---------+---------+
                              |
                     Audio Windows
                              |
                             gRPC
                              |
                              v
                    +-------------------+
                    |   AKSHAT'S ML     |
                    |  INFERENCE LAYER  |
                    +---------+---------+
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          AASIST           PROSODY          ECAPA
             |                |                |
             v                v                v
       artifact_score   prosody_score   speaker_mismatch_score
             |                |                |
             +----------------+----------------+
                              |
                              v
                    +-------------------+
                    | SANJEEVANI FUSION |
                    |     LightGBM       |
                    +---------+---------+
                              |
                              v
                         risk_score
                              |
                              v
                         risk_level
                              |
                              v
                    +-------------------+
                    | BACKEND / GATEWAY |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |  YOSHITA FRONTEND |
                    +-------------------+

                    MANAV INFRASTRUCTURE
              Docker / Compose / Deployment /
                 Health / Logs / Monitoring