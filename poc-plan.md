# PoC Plan: llmsearchindex

## Project Classification
- **Type:** rag
- **Key Technologies:** Python, FAISS (faiss-cpu), sentence-transformers (all-MiniLM-L6-v2), PyTorch, Streamlit, HuggingFace Hub, PCA + Binary Quantization, NumPy, scikit-learn
- **ODH Relevance:** This is a fully local, internet-scale search index designed specifically for LLM RAG applications. It demonstrates how to provide external context to LLMs without external API calls at query time — a core use case for Open Data Hub's model serving and RAG capabilities. The FAISS-based vector search and sentence-transformer embeddings align directly with ODH's support for ML inference workloads.

## PoC Objectives
What we want to prove:
1. The Streamlit demo application can be containerized and deployed to OpenShift, serving as a web-accessible search interface
2. The LLMSearchIndex library successfully downloads its pre-built index (~10GB) from HuggingFace Hub at startup
3. The sentence-transformers embedding model loads and runs correctly for query encoding
4. Search queries against the 203M web page index return relevant results within acceptable latency
5. The PCA dimensionality reduction (384d → 64d) and binary quantization pipeline works end-to-end in a containerized environment

## Infrastructure Requirements
- **Inference Server:** none (the library handles its own embedding and search)
- **Vector Database:** in-memory (FAISS index downloaded from HuggingFace Hub and loaded into RAM)
- **Embedding Model:** sentence-transformers/all-MiniLM-L6-v2 (downloaded automatically by sentence-transformers)
- **GPU Required:** No (CPU inference supported; GPU optional)
- **Persistent Storage:** 15Gi PVC recommended — the FAISS index is ~10GB and the embedding model is ~90MB. A PVC allows caching these so restarts don't re-download. The HuggingFace cache directory (`~/.cache/huggingface`) and any local index files should be stored on the PVC.
- **Resource Profile:** large (6GB+ RAM required for the FAISS index, plus CPU for embedding)
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: Health Check
- **Description:** Verify the Streamlit application is running and healthy
- **Type:** http
- **Endpoint:** `/_stcore/health`
- **Input:** None
- **Expected:** Returns 200 OK, indicating the Streamlit server is up
- **Timeout:** 300 seconds (first startup may take time to download the index)

### Scenario 2: UI Loads
- **Description:** Verify the main Streamlit page renders successfully
- **Type:** http
- **Endpoint:** `/`
- **Input:** None
- **Expected:** Returns 200 OK with HTML content
- **Timeout:** 60 seconds

### Scenario 3: Search Functionality
- **Description:** Execute a search query using the LLMIndex library directly inside the running container to verify the full pipeline (embedding → PCA → binary quantization → FAISS search → result fetching)
- **Type:** exec
- **Input:** `python -c "from llmsearchindex import LLMIndex; idx = LLMIndex(); results = idx.search('who invented sliced bread', top_k=3); assert len(results) > 0; print('SUCCESS:', len(results), 'results'); print(results[0].get('text','')[:200])"`
- **Expected:** Exits 0, prints at least 1 result with relevant text about sliced bread or Otto Rohwedder
- **Timeout:** 600 seconds (index download may take several minutes on first run)

## Dockerfile Considerations

This is a **Streamlit web application** that listens on port 8501.

- **Base image:** Use `python:3.11-slim` or similar. The project needs PyTorch (CPU), FAISS (CPU), and sentence-transformers.
- **Dependencies:** Install from `requirements.txt` first, then install the `llmsearchindex` package itself (either via `pip install .` or by copying the package directory).
- **EXPOSE 8501** — The Streamlit app listens on this port.
- **ENTRYPOINT/CMD:** `streamlit run streamlit_app.py --server.port=8501 --server.address=0.0.0.0 --server.headless=true`
- **Copy all files** from the repo root — `streamlit_app.py`, `search.py`, `llmsearchindex/` package directory, `requirements.txt`, `pyproject.toml`.
- **Note on size:** The container image itself will be moderate (~2-3GB with PyTorch CPU). The large FAISS index (~10GB) is downloaded at runtime from HuggingFace Hub, NOT baked into the image.
- **Cache directory:** Consider setting `HF_HOME=/data/hf_cache` or similar environment variable to direct HuggingFace downloads to a PVC-mounted path for persistence across restarts.
- **Non-root user:** Create and use a non-root user for OpenShift compatibility.

## Deployment Considerations

- **Deployment model:** Deploy as a **Kubernetes Deployment** with 1 replica. This is a long-running Streamlit web server.
- **Service:** Create a **Service on port 8501** to expose the Streamlit UI.
- **PVC:** Mount a 15Gi PVC at `/data` (or `/home/appuser/.cache`) to cache the HuggingFace model and FAISS index downloads. Set `HF_HOME` environment variable to point to this path.
- **Resource requests/limits:** Request at least 6Gi RAM and 2 CPU cores. The FAISS index is loaded entirely into memory. Limit to 8Gi RAM and 4 CPU cores.
- **Startup time:** The first startup will be slow (5-10 minutes) as it downloads the ~10GB FAISS index and the embedding model from HuggingFace Hub. Set `initialDelaySeconds` on readiness/liveness probes accordingly (e.g., 300 seconds).
- **Liveness probe:** HTTP GET on `/_stcore/health` port 8501, with a generous `initialDelaySeconds` of 300 and `periodSeconds` of 30.
- **No GPU required** — CPU inference is fully supported.
- **No external API keys required** — All search is local. The index and model are downloaded from public HuggingFace repositories.
- **Network access:** The pod needs outbound internet access to download from `huggingface.co` on first startup. After the index is cached on the PVC, it can run without internet.
- **Test via HTTP:** Send requests to the Streamlit health endpoint and UI endpoint. For deeper functional testing, use `kubectl exec` to run a Python search query inside the pod.