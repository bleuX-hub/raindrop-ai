# Raindrop AI Companion – Monorepo

> **Codename:** `raindrop-ai` (rename later)
>
> Minimal, local‑first wearable AI stack inspired by the conversation above.  This repo holds **all** layers – firmware, mobile app, and ML models – in a single monorepo to keep iterations tight.

---

## 1. High‑Level Architecture

```
├── firmware/          # nRF54 SDK project – BLE audio bridge & haptics
├── mobile/            # iOS + Android apps (Swift + Kotlin Multiplatform)
├── ml/                # LLM, STT, TTS assets & scripts
│   ├── models/        # downloaded + quantised weights live here (git‑ignored)
│   └── notebooks/     # exploration & fine‑tuning
├── cloud/             # optional API / RAG endpoint (FastAPI + Supabase)
└── docs/              # design briefs, regulatory, BOM, etc.
```

### Data Flow

1. **Wake‑word** picked up on brooch → streamed to phone via BLE.
2. **STT** (`whisper.cpp`) converts audio → text.
3. **Intent + mood** classifier parses text & prosody.
4. **LLM** (Llama‑3‑8B‑Instruct Q4\_K\_M) generates response with persona & RAG memories.
5. **TTS** renders voice (Apple Neural or Coqui XTTS).
6. Phone transmits audio/haptic cue → brooch.

All critical inference runs *on the phone*; cloud used only when explicitly enabled.

---

## 2. Base Model Selection

| Candidate                 | Pros                                                                                     | Cons                                                     |
| ------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **Apple 3 B** (CoreML)    | Ultra‑efficient on iOS; tiny footprint                                                   | iOS‑only, weaker reasoning                               |
| **Mistral‑7B**            | Strong reasoning <8 GB RAM                                                               | Slightly slower on Android                               |
| **Llama‑3‑8B‑Instruct** ✅ | Best accuracy of sub‑10 B models; clean license; CoreML + llama.cpp toolchains available | Needs \~3.7 GB RAM (4‑bit) – acceptable on modern phones |

> **Decision:** Use **Llama‑3‑8B‑Instruct**, quantised to `Q4_K_M` via `llama.cpp`, as the cross‑platform default.  Provide Apple‑3B option for ultra‑low‑power mode.

---

## 3. Quick‑start (Mac / Linux)

```bash
# 0. Clone
$ git clone https://github.com/your‑org/raindrop‑ai.git && cd raindrop‑ai

# 1. Install toolchain
$ brew install cmake protobuf rust python@3.11
$ pip install -r ml/requirements.txt  # whisper.cpp, llama.cpp Python wheels, sentence‑transformers

# 2. Fetch base model (~5 GB)
$ ./ml/scripts/get‑llama3.sh  # pulls from Meta release, converts to GGUF

# 3. Quantise to 4‑bit & test
$ ./ml/scripts/quantise‑run.sh "Hello, Raindrop!"
```

### iOS (CoreML) Build

1. `cd ml/coreml && make convert‑llama3` – converts 8B to CoreML FP16.
2. Open `mobile/ios/Raindrop.xcworkspace` in Xcode 15+.
3. Run on device; first launch copies model to app sandbox (\~4 GB).

### Android JNI Build

1. NDK r26 + CMake.
2. `./mobile/android/gradlew installDebug` – Gradle script compiles llama.cpp JNI.

---

## 4. Memory & RAG Layer

* Lightweight `Chroma` vector store embedded (SQLite backend).
* `ml/rag/` contains:

  * `embed.py` – MiniLM sentence embeddings (Quantised, 120 MB).
  * `memory.py` – CRUD ops, stores max 10k entries.

---

## 5. Development Roadmap

| Milestone | Target                                         | Owner         |
| --------- | ---------------------------------------------- | ------------- |
| **M0**    | Voice loop MVP (wake → STT → LLM → TTS) on iOS | @mobile‑dev   |
| **M1**    | BLE audio bridge FW + haptic pilot             | @firmware‑dev |
| **M2**    | Emotion classifier fused                       | @ml‑research  |
| **Alpha** | 100‑user TestFlight + 3‑D printed pendant      | @pm           |

---

## 6. Contributing

* `git checkout -b feat/<ticket>`
* Commit with Conventional Commits (`feat:`, `fix:`, etc.)
* Run `pre‑commit` hooks (black, flake8, clang‑format)

---

## 7. License

MIT for code; model weights follow original license (Meta Llama 3, CC‑BY‑NC‑4.0).

---

## 8. Getting Started if You're *Not* a Developer

So you love the concept but "git clone" sounds like Klingon—no problem. Here’s the **lowest‑friction way to move forward without writing a single line of code**:

1. **Create a GitHub account** (free) → click **Fork** on this repo once it’s pushed. That gives you your own copy in the cloud—no terminal needed.
2. **Invite a technical collaborator**: Settings ▸ Collaborators ▸ add their GitHub handle. Tell them to look at the `README` roadmap.
3. **Spin up GitHub Codespaces** (green "Code" ▸ **Codespaces** tab ▸ +). It pre‑installs all compilers & Python—zero local setup.
4. **Run the Quick‑start script in the browser:**

   ```bash
   ./ml/scripts/quick_demo.sh "Hi there!"
   ```

   The script grabs a tiny demo model and speaks a reply—just to prove the pipeline.
5. **Project‑manage, don’t program**: use GitHub Issues to break features into tasks (e.g., "Add BLE Audio Stub", "Design Wake‑word Logo").
6. **Budget for expert help**: expect 1 senior mobile dev + 1 embedded dev part‑time for 8–12 weeks to hit Alpha.
7. **Use the README as the single source of truth**—update it after each sprint so you’re steering, not coding.

> **Tip:** You can still peek at progress anytime—Codespaces lets you click around files and even run the iOS app in a web‑based simulator.

---

## 9. TODO (open questions)

* [ ] Decide on final wake‑word engine (Snowboy successor vs. Picovoice)
* [ ] Battery usage telemetry API
* [ ] GDPR data‑export endpoint

> **Next steps:**
>
> 1. Fork this template to GitHub, set CI via GitHub Actions (build iOS/Android sample, run unit tests).
> 2. Slack #raindrop‑dev for stand‑up notes.
