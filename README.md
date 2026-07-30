# SmartVision

<p align="center"><strong>A Turkish voice-first visual assistant engineered for continuity, privacy, and meaningful scene understanding.</strong></p>

<p align="center">
<img src="https://img.shields.io/badge/Android-Kotlin-7F52FF?logo=kotlin&logoColor=white" alt="Kotlin"> <img src="https://img.shields.io/badge/Edge_AI-LiteRT%20%2F%20TensorFlow%20Lite-FF6F00?logo=tensorflow&logoColor=white" alt="LiteRT"> <img src="https://img.shields.io/badge/API-Python%20%2B%20FastAPI-009688?logo=fastapi&logoColor=white" alt="FastAPI"> <img src="https://img.shields.io/badge/Data-PostgreSQL-4169E1?logo=postgresql&logoColor=white" alt="PostgreSQL"> <img src="https://img.shields.io/badge/Hosted_on-Render-46E3B7?logo=render&logoColor=white" alt="Render">
</p>

SmartVision is a research-driven edge-cloud assistant for visually impaired people. It converts Turkish voice commands into actions, reads printed text aloud, and provides spoken environmental descriptions from a mobile or external camera module. Essential interaction remains on-device; detailed scene understanding is delegated to the cloud only when higher compute provides a clear benefit.

## Design Philosophy: Accessibility Over Aesthetics

SmartVision's interface is intentionally simple. In an assistive product, visual ornament, dense navigation, and interaction novelty can create friction rather than value. The design is therefore optimized for rapid task access, high-contrast presentation, screen-reader compatibility, and low cognitive load—not for decorative complexity.

This minimalism is an engineering decision, not an absence of effort. Large, unambiguous controls and a reduced interaction surface help users recognize the application state and act quickly. The interface prioritizes the user journey that matters: listen, capture or select an image, understand the result, and receive it through speech.

<p align="center">
  <img src="docs/images/01-main-listening-interface.jpeg" width="250" alt="SmartVision main listening interface">
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img src="docs/images/02-accessibility-settings.jpeg" width="250" alt="SmartVision accessibility settings">
</p>

## The problem and the hybrid solution

Cloud-only assistants can generate rich descriptions but become fragile when connectivity fails. Fully local solutions preserve privacy and continuity, but mobile hardware is not designed for every multimodal reasoning task. SmartVision makes this trade-off explicit.

| User need | Execution location | Engineering rationale |
|---|---|---|
| Turkish speech-to-text | Android edge | Keeps speech on-device and eliminates network latency. |
| Intent classification | Android edge | Compact LiteRT Bi-LSTM supports offline command handling, including out-of-scope inputs. |
| OCR and spoken feedback | Android edge | Printed-text reading works without internet; OCR images are not uploaded. |
| External image capture | Local Wi-Fi | ESP32/M5Stack camera communicates directly with the phone; targeted for future wearable form factor. |
| Open-ended scene description | Cloud | A multimodal model provides richer contextual reasoning than the mobile stack targets. |

```mermaid
flowchart LR
 U["User"] -->|"Turkish voice"| A["Android edge client"]
 E["ESP32 / M5Stack camera"] -->|"Local Wi-Fi image"| A
 A -->|"Offline"| S["Vosk STT"] --> I["LiteRT Bi-LSTM"]
 A -->|"Offline"| O["ML Kit OCR"] --> T["Android Text-to-Speech"]
 A -->|"Scene request only\n≤800 px · JPEG 70%"| F["FastAPI on Render"]
 F --> L["Gemini 3.1 Flash Lite"]
 F --> P["PostgreSQL telemetry"]
 L --> T
```

## Edge intelligence

The Android client uses Vosk (`vosk-model-small-tr-0.3`) for offline Turkish speech recognition, a custom LiteRT/TensorFlow Lite Bi-LSTM for command classification, Google ML Kit Text Recognition for on-device OCR, and Android Text-to-Speech for Turkish feedback. The intent model handles seven actionable commands plus an `OUT_OF_SCOPE` class, so unsupported requests are identified by a learned decision boundary instead of a fixed confidence threshold. Images over 1024 pixels are proportionally reduced before OCR to control mobile memory use.

The original 1D-CNN classifier was replaced with a Bi-LSTM architecture after a data leakage issue was identified in the training pipeline. The earlier model's augmentation variants leaked across training and validation splits, inflating accuracy. The current model uses `StratifiedGroupKFold` to partition data at the root-pattern level and removes validation/test sentences whose normalized core pattern exactly overlaps with a training pattern.

The original nine classes were reduced to seven actionable commands: `READ_TEXT`, `TAKE_PHOTO`, `PICK_GALLERY`, `FETCH_ESP32`, `STOP`, `REPEAT`, and `HELP`. `OCR_CAPTURE` was merged into `READ_TEXT` because both trigger the same action. An eighth `OUT_OF_SCOPE` class is trained directly to identify unrelated daily-life requests such as weather, greetings, music, alarms, and navigation.

The model comprises TextVectorization (2,500-token vocabulary, sequence length 15), a 64-dimensional mask-aware embedding layer, 40% dropout, a 64-unit bidirectional LSTM, a 32-unit L2-regularized ReLU dense layer, a second 40% dropout layer, and an eight-class softmax output. It has 230,440 parameters and is deployed as a 262 KB LiteRT/TensorFlow Lite model.

> Text reading is completely offline: OCR runs on the Android device, so the reading flow avoids cloud latency and keeps the captured text image private.

<p align="center"><img src="docs/images/04-gallery-ocr-reading.jpeg" width="250" alt="Gallery OCR reading flow"></p>

<p align="center">
  <img src="docs/images/confusion_matrix_v4_en.png" width="700" alt="Bi-LSTM intent classifier confusion matrix">
  <br><br>
  <img src="docs/images/training_curves_v4_en.png" width="700" alt="Bi-LSTM intent classifier training curves">
</p>

## Backend and cloud architecture

The online layer is a FastAPI service actively hosted on **Render**, backed by **PostgreSQL** for usage and performance telemetry. It has a deliberately narrow responsibility: authorize scene-analysis requests, invoke the multimodal model, return a concise Turkish description, and record operational metadata.

The client first checks connectivity, bounds the image to 800 pixels, compresses it as JPEG at 70% quality, and sends multipart data. FastAPI authenticates requests through `X-API-Key`; the application key, Gemini API key, database URL, and database-ping token are supplied as environment variables. The service calls Google's `gemini-3.1-flash-lite` with instructions that prioritize people, motion, hazards, obstacles, doors, stairs, vehicles, and spatial context. The free tier provides 15 RPM, 250K TPM, and 500 RPD; paid tiers can be adopted as the project scales.

| Concern | Design response |
|---|---|
| API access | `X-API-Key` gate checked against a server-side secret. |
| Resource control | Async semaphore limits scene analysis to two concurrent requests. |
| Upstream resilience | 30-second multimodal timeout and categorized error handling. |
| Client abandonment | Disconnection checks occur before and after model work. |
| Observability | Structured logs and PostgreSQL telemetry record duration, image size, tokens, outcome, errors, and model name. |

> The scene-analysis flow sends an optimized image to Gemini 3.1 Flash Lite only after an explicit online request, then returns its contextual Turkish description for spoken feedback.

<p align="center"><img src="docs/images/03-camera-scene-analysis.jpeg" width="250" alt="Camera scene analysis"></p>

### Deployment & Telemetry

The FastAPI application is actively deployed on **Render**. It records operational usage metadata in **PostgreSQL**, making latency, token usage, image payload size, action outcomes, and error categories observable for research evaluation and service monitoring.

<p align="center">
  <img src="docs/images/05-render-backend-dashboard.png" width="700" alt="Render backend dashboard">
  <br><br>
  <img src="docs/images/06-postgresql-telemetry-logs.png" width="700" alt="PostgreSQL telemetry logs">
</p>

## Data and performance

Measurements below are reported in the academic paper for the experimental setup.

| Metric | Reported value | Interpretation |
|---|---:|---|
| Supported classes | 8 | Seven actionable commands plus `OUT_OF_SCOPE`. |
| Data split method | StratifiedGroupKFold + pattern purification | Root-pattern-level split and removal of exact normalized-pattern overlaps prevent data leakage. |
| Model architecture | Bi-LSTM | Bidirectional context captures word-order variations. |
| Model parameters | 230,440 | Optimized for edge deployment. |
| LiteRT model size | 262 KB | Compact mobile footprint. |
| Intent accuracy | 78.85% | Evaluated on the purified test set. |
| Weighted F1-score | 80.37% | Accounts for class imbalance across the eight classes. |
| Macro F1-score | 78.62% | Gives each class equal weight. |
| Offline intent classification | <100 ms | Local command routing without a network round trip. |
| Offline OCR | ~843 ms | No external network round trip. |
| Cloud scene analysis | ~1,349 ms | Multimodal reasoning offloaded to Gemini 3.1 Flash Lite. |
| Avg. tokens per scene request | 1,258 | Measured across 12 consecutive test requests. |
| Avg. image size per request | 29.6 KB | After client-side JPEG compression. |

| Dimension | Local OCR / intent | Cloud scene analysis |
|---|---|---|
| Network dependency | None | Internet required |
| Optimization goal | Privacy, continuity, response speed | Rich semantic understanding |
| Externally transmitted data | None | Optimized image only for a requested scene analysis |
| Compute target | Android device | Render-hosted FastAPI + Gemini 3.1 Flash Lite |

The 78.85% test accuracy is measured after group-based splitting and dataset purification, which prevent exact normalized command patterns from appearing in both training and evaluation data. A separate, stricter unseen-root-family experiment achieved 38.84% accuracy and 32.96% macro F1, highlighting the need for future Turkish pretrained embeddings such as fastText or BERT.

## Code quality highlight: defensive local routing

The classifier is designed to recognize both supported actions and out-of-scope inputs locally, without introducing a network dependency in the command path.

```kotlin
override fun classify(text: String): IntentClassifierResult {
    val cleaned = text.trim()
    if (cleaned.isEmpty()) {
        return IntentClassifierResult.Success(IntentAction.UNKNOWN)
    }

    return primary?.classify(cleaned)
        ?: IntentClassifierResult.Success(IntentAction.UNKNOWN)
}
```

This compact structure uses normalization and guard clauses to reduce nesting, keeps the primary model replaceable behind an interface, and makes failure behaviour explicit.

## Academic status and conclusion

SmartVision was supported through the **TÜBİTAK 2209-A University Students Research Projects Support Program**. The project was subsequently developed independently, and its technical study is currently being formatted for academic conference submission.

SmartVision demonstrates an accessibility architecture that does not force a choice between offline resilience and capable AI: the edge owns private, immediate interaction, while cloud multimodal reasoning is invoked only where it delivers a meaningful benefit. Future work includes broader user validation, tactile input for noisy environments, expanded command coverage, Turkish pretrained embeddings, and compact local multimodal models for offline scene understanding.

---

**Portfolio repository notice:** This public repository presents the SmartVision case study and research outcomes. Production source code, model assets, datasets, credentials, and deployment configuration remain private.
