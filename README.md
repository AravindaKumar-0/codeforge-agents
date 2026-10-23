CI/CD Pipeline Verification - 23 October 2026

# 🔨 CodeForge Agents

**Watch AI agents debug, review and optimize your code — live, with the hood open.**

Built for the CMRIT FDP *"Agentic AI: Development of Intelligent Agents with Modern AI Frameworks."*

Paste code or ask a question → a **Router** classifies it → **RAG** retrieves from a local knowledge base → a deterministic **Tool** runs (Python sandbox, YAML/K8s validator, Java checker) → **Reviewer**, **Optimizer** and **Synthesizer** agents reason over the result — every step visible in the UI.

> Works **with or without** a cloud key. The LLM uses a **fallback chain**: 🟢 Groq → 🟡 local Ollama → 🔴 Tools-Only Mode. As long as one backend is reachable, the app answers — and it never crashes when none are.
>
> No key on the command line required — **paste a Groq key right in the sidebar** (it stays in your session, never logged), or supply it via `.env` / a Kubernetes Secret.

---

## 🆕 Recent improvements

- **📄 Bring your own knowledge** — upload a `.txt`/`.md` file or paste text in the sidebar and the agents cite *your* standards/notes alongside the built-in knowledge base. Per session, in-memory, nothing written to disk. ([details](#-bring-your-own-knowledge-per-session))
- **🔑 API key from the UI** — connect Groq by pasting the key in the app, not just on the command line. The `.env` / Secret path still takes precedence, so Docker and k8s are unchanged.
- **🐳 Faster container start** — the ONNX embedding model and Chroma index are now baked into the image **as the runtime user**, so containers no longer re-download the 79 MB model on every start. Fully offline-ready.

---

## 🏗️ Architecture

```
             ┌──────────────────────────────────────────────┐
             │                Streamlit UI                  │
             │  Router │ RAG │ Tool │ Reviewer │ Optimizer  │
             │                  │ Synthesizer               │
             └──────┬───────────────────────────┬───────────┘
                    │                           │
         ┌──────────▼──────────┐     ┌──────────▼──────────┐
         │   RAG + Tools        │     │   LLM Fallback Chain │
         │  ChromaDB (ONNX      │◄────┤  Groq  → Ollama →    │
         │  MiniLM embeddings)  │ctx  │  Tools-Only          │
         │  python/yaml/java    │     │  llama-3.3-70b /     │
         │  deterministic tools │     │  local mistral       │
         └──────────┬──────────┘     └──────────┬──────────┘
                    │                           │ (cloud, optional)
             ┌──────▼───────┐            ┌──────▼───────┐
             │ knowledge/   │            │   Groq API   │
             │ *.md  index  │            └──────────────┘
             └──────────────┘

Git push ──► GitHub repo ──► ArgoCD (auto-sync) ──► Kubernetes
                                  ▲                    │
                                  └── self-heal ◄──────┘
docker build ──► Docker Hub (appars/codeforge-agents)
```

**The agentic part:** each request flows through a pipeline of specialised agents. The Reviewer judges correctness against the deterministic tool result (it never guesses whether code ran), the Optimizer proposes safe improvements, and the Synthesizer merges everything — tool output, retrieved knowledge, both agents' notes — into one streamed answer. The full trace is expandable in the UI.

---

## 🚀 Local Development

```bash
git clone https://github.com/appars/codeforge-agents
cd codeforge-agents

python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Build the knowledge index ONCE (no API key needed — embeddings are local).
# Commit the resulting chroma_db/ so cloud deploys start instantly.
python -m rag.ingest

streamlit run app.py
# → http://localhost:8501
```

Click a **Demo Scenario** card to watch the full pipeline run on a planted bug.

### Run tests

```bash
pip install pytest
pytest -q
```

---

## 🔑 LLM Configuration (the fallback chain)

The app tries backends **in order** on every request, so a rate limit or a missing key degrades gracefully instead of failing:

| Mode        | Indicator | When it runs                          | Features                                              |
| ----------- | --------- | ------------------------------------- | ----------------------------------------------------- |
| Groq        | 🟢         | `GROQ_API_KEY` set and under quota    | Full pipeline on `llama-3.3-70b-versatile` (fast)     |
| Ollama      | 🟡         | Groq unavailable, local Ollama up     | Full pipeline on a local model (offline, no key)      |
| Tools-Only  | 🔴         | No LLM reachable                      | Sandbox + YAML/Java validation + RAG, no agents       |

Get a free Groq key at <https://console.groq.com/keys>. Resolution order:

1. `.env` file → `cp .env.example .env`, set `GROQ_API_KEY=gsk_...`
2. Environment variable → `export GROQ_API_KEY=gsk_...`
3. Streamlit Cloud → **Settings → Secrets** (see below)
4. **App sidebar** → paste the key into the 🔑 *Connect an LLM* field — stored in that browser session only, never logged or saved (handy when an attendee brings their own key)
5. Nothing → 🟡 Ollama if running, else 🔴 Tools-Only

A key from sources 1–3 takes precedence, so the sidebar field only appears when no key is configured.

For the best **local** experience, install [Ollama](https://ollama.com) and pull a coder model:

```bash
ollama pull qwen2.5-coder      # strong at debugging; set OLLAMA_MODEL=qwen2.5-coder
ollama serve
```

Switch the chain order any time with `LLM_PROVIDER` (`auto` | `groq` | `ollama`) in `.env`.

---

## 📄 Bring your own knowledge (per session)

Beyond the built-in `knowledge/*.md` base, anyone can add their **own** material at runtime from the sidebar — a team style guide, a grading rubric, lab guidelines — and watch the agents cite it.

- **Upload** a `.txt` or `.md` file (or several), **or paste** text directly — no file or markdown needed.
- It is chunked and embedded with the **same** model as the baked index, so retrieval ranks your chunks against the built-in ones by relevance. Hits are badged 📄 *your upload* vs 📚 *base* in the RAG trace.
- Stored **in-memory, per browser session** (`chromadb.EphemeralClient`): isolated between users, written to no disk, and gone when the tab closes. Nothing leaks across sessions or, in multi-replica Kubernetes, across pods.
- Safeguards: tolerant decoding (Word/Windows files), 1 MB per file, ~200 chunks per session, content-hash dedup, and a **Clear my knowledge** button.

> Plain text is chunked by paragraph; markdown with `## ` headings is chunked by section. The built-in knowledge base is untouched — uploads live only in your session.

---

## 🐳 Docker

```bash
# Build (the image bakes the knowledge index AND the ONNX model at build
# time, as the non-root runtime user — no key needed, no download at start)
docker build -t appars/codeforge-agents:latest .

# Run (Tools-Only Mode — no key; or paste a Groq key in the sidebar)
docker run -p 8501:8501 appars/codeforge-agents:latest

# Run (Groq Mode) — prefer --env-file over -e: a key passed with -e is
# visible in shell history and in `docker inspect`
docker run -p 8501:8501 --env-file .env appars/codeforge-agents:latest

# Push to Docker Hub
docker login
docker push appars/codeforge-agents:latest
```

Image highlights: `python:3.11-slim`, non-root user (UID 10001), layer-cached deps, ONNX embeddings (no PyTorch), built-in `HEALTHCHECK` on Streamlit's `/_stcore/health`. The index and embedding model are baked **as UID 10001**, so the container reads them from the same home it runs as — no 79 MB download on startup, works on an airgapped network.

---

## ☁️ Deploy to Streamlit Cloud

Streamlit Cloud runs on a small (~1 GB) container and **does not use the Dockerfile**, so it cannot build the index at boot without running out of memory. The fix is to **commit a pre-built index**:

```bash
python -m rag.ingest          # builds chroma_db/ with the real ONNX model
git add chroma_db .gitignore
git commit -m "Commit pre-built knowledge index for cloud boot"
git push
```

Then on share.streamlit.io:

1. New app → point at `appars/codeforge-agents`, branch `main`, `app.py`.
2. **Settings → Secrets** → add (TOML format):
   ```toml
   GROQ_API_KEY = "gsk_your_key_here"
   ```
3. Deploy. The app boots instantly (index already present) and answers via Groq.

> Ollama isn't reachable from Streamlit Cloud, so the chain there is effectively Groq → Tools-Only. Keep `GROQ_API_KEY` valid.

---

## 🖥️ Rancher Desktop Setup

1. Install [Rancher Desktop](https://rancherdesktop.io/) → enable **Kubernetes**.
2. Verify: `kubectl get nodes` shows the node `Ready`.
3. NodePort services are reachable at `localhost:<nodePort>`.

---

## ☸️ Kubernetes Deployment (manual)

```bash
# Create the Groq secret FIRST (never put the key in a YAML file).
# Skip this to run in Tools-Only / Ollama mode.
kubectl create secret generic codeforge-secrets \
    --from-literal=GROQ_API_KEY='gsk_xxx'

kubectl apply -f k8s/
kubectl get pods -l app=codeforge-agents      # 2 replicas Running

# Access (NodePort)
open http://localhost:30851
# or: kubectl port-forward svc/codeforge-agents 8501:8501
```

Manifests follow the project's own `knowledge/team_standards.md` rules: team label, resource requests/limits, pinned image tag, two replicas, secret injected from the cluster (never inline), non-root securityContext, readiness + liveness probes on `/_stcore/health`.

---

## 🔁 GitOps with ArgoCD

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# UI access
kubectl port-forward svc/argocd-server -n argocd 8080:443
# initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Register the app
kubectl apply -f argocd/application.yaml
```

ArgoCD now watches `github.com/appars/codeforge-agents` (path `k8s/`, branch `main`) with **auto-sync + self-heal + prune**.

**Live demo moment 🎬:** edit `k8s/deployment.yaml` in GitHub (`replicas: 2 → 3`), commit, and watch ArgoCD detect, sync, and roll out — no `kubectl`. Then `kubectl scale deploy codeforge-agents --replicas=1` and watch self-heal revert it. That's GitOps.

---

## 🧪 Health Check

```bash
curl -fsS http://localhost:8501/_stcore/health   # → "ok"
```

Used by the Docker `HEALTHCHECK` and both Kubernetes probes.

---

## 🛠️ Troubleshooting

| Symptom                            | Fix                                                                                       |
| ---------------------------------- | ----------------------------------------------------------------------------------------- |
| Streamlit Cloud `connection refused` at boot | Commit a pre-built index: `python -m rag.ingest` then push `chroma_db/`         |
| 🔴 Tools-Only despite a key         | Check `GROQ_API_KEY` (env / `.env` / Secrets); on cloud it must be TOML in **Secrets**    |
| 🟡 Ollama mode locally              | Expected when Groq is rate-limited (429) — the chain fell back; or start `ollama serve`   |
| `ImagePullBackOff`                 | Image not pushed / wrong name → `docker push appars/codeforge-agents:latest`              |
| Pod `OOMKilled`                    | Raise the memory limit in `k8s/deployment.yaml` (default 1Gi)                             |
| Probes failing                     | Check `kubectl logs`; Streamlit needs ~10–20 s to boot                                    |
| ArgoCD `Unknown`/`ComparisonError` | Repo URL/branch/path wrong, or repo is private (add repo credentials in ArgoCD)           |
| NodePort unreachable               | `kubectl port-forward svc/codeforge-agents 8501:8501`                                      |
| First answer slow                  | Ollama loads the model into RAM on first call; later calls are fast                       |
| Container re-downloads 79 MB at start | Old image baked the model as root but ran as `forge`. Rebuild with the current Dockerfile (bakes as UID 10001). |

---

## 📸 Screenshots

|                                                                 |                                                              |
| --------------------------------------------------------------- | ------------------------------------------------------------ |
| ![Pipeline](assets/screenshot-pipeline.png)                     | ![Agent trace](assets/screenshot-agents.png)                 |

*(placeholders — add after first run)*

---

## 📁 Project Structure

```
codeforge-agents/
├── app.py                     # Streamlit UI (pipeline, sidebar, modes)
├── core/
│   ├── config.py              # Central config, secret resolution
│   ├── llm.py                 # LLM fallback chain (Groq → Ollama → none)
│   ├── router.py              # LLM + keyword intent/language classifier
│   ├── memory.py              # Conversation memory window
│   ├── scenarios.py           # Demo scenario cards
│   └── ui.py                  # Design system (the "forge" theme)
├── agents/
│   ├── base.py                # Shared agent contract
│   ├── reviewer.py            # Correctness / risk review
│   ├── optimizer.py           # Safe improvement suggestions
│   └── synthesizer.py         # Streams the final answer
├── tools/
│   ├── python_runner.py       # Sandboxed subprocess execution
│   ├── yaml_validator.py      # YAML + Kubernetes structure checks
│   ├── java_checker.py        # Java heuristics
│   └── extract.py             # Fenced code-block extraction
├── rag/
│   ├── embedder.py            # ONNX MiniLM embeddings (no torch)
│   ├── chunking.py            # Markdown (per-section) + plain-text chunking
│   ├── ingest.py              # Build the ChromaDB index
│   ├── retrieve.py            # Retrieval + relevance threshold + session-KB merge
│   └── user_kb.py             # Session-scoped "bring your own knowledge" store
├── knowledge/                 # *.md knowledge base (debug/optimize/standards)
├── chroma_db/                 # Pre-built index (committed for cloud boot)
├── tests/                     # Unit + fallback tests
├── k8s/                       # deployment + service (NodePort)
├── argocd/application.yaml    # GitOps app (auto-sync, self-heal, prune)
├── Dockerfile                 # python:3.11-slim, non-root, healthcheck
├── requirements.txt
└── .env.example
```

---

## 📜 License

MIT — built for teaching. Reuse freely in your FDP sessions.
