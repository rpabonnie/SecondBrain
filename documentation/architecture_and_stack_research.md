# Local RAG "Second Brain" - Architecture & Stack Research

**Date:** November 22, 2025
**Role:** Principal Software Architect & AI Systems Engineer
**Subject:** Architectural Blueprint and Technology Selection for Local Notion-based RAG System

---

## Executive Summary

This document outlines the architectural strategy for building a "Second Brain" RAG system constrained by local hardware (RTX 4060, 16GB RAM) and a zero-cost requirement. The system aims to ingest data from a Notion Workspace and provide a chat interface for retrieval.

Given the hardware constraints—specifically the 16GB system RAM and 8GB VRAM—efficiency is the primary driver for all technology decisions. The system must balance a heavy OS (Windows 11) with memory-intensive AI operations.

---

## 1. Architectural Design & Patterns

### Comparison of Patterns

#### **Modular Monolith**
*   **Concept:** A single deployable unit (executable/process) where code is organized into distinct modules (Ingestion, Retrieval, UI) that communicate via in-memory function calls.
*   **Pros:** Lowest operational complexity. No network overhead between services. Shared memory usage (efficient for low-RAM systems).
*   **Cons:** Tighter coupling; a crash in the ingestion worker could bring down the chat interface.
*   **Suitability:** High. It minimizes the overhead of running multiple containers or processes, which is crucial for a 16GB RAM machine.

#### **Clean Architecture (Onion/Hexagonal)**
*   **Concept:** Strict separation of concerns where the "Domain" (Core Logic) is at the center, surrounded by "Application" (Use Cases), and finally "Infrastructure" (Notion API, Vector DB, UI).
*   **Pros:** Highly testable. Decouples the volatile Notion API and specific Vector DB implementation from the core logic. Allows swapping the DB later without rewriting business logic.
*   **Cons:** Higher boilerplate code.
*   **Suitability:** High. This is a code organization pattern, not a deployment pattern. It can be applied *within* a Modular Monolith.

### **Recommendation: The "Async Modular Monolith"**

For a local Windows machine with limited resources, a **Modular Monolith** structured using **Clean Architecture** principles is the optimal choice.

*   **Deployment:** A single Python application (or .NET executable) that spawns a background thread/process for the "Ingestion Worker".
*   **Communication:** The Chat Interface and Ingestion Worker share access to the Vector Database (Infrastructure layer).
*   **Why:** Running microservices (e.g., a separate Docker container for API, one for Worker, one for Frontend) adds significant RAM overhead (Docker Desktop alone can consume 2GB+). A single optimized process is best for your hardware.

---

## 2. Technology Stack Selection

### **Language: Python**
*   **Verdict:** **Best for Project.**
*   **Justification:** While C#/.NET is superior for Windows background services, the RAG ecosystem in Python (LlamaIndex, LangChain) is years ahead in terms of "Notion-specific" tooling.
    *   **LlamaIndex** has robust, pre-built Notion readers that handle the complex block structure of Notion pages better than any .NET equivalent.
    *   **Development Speed:** AI coding assistants are most proficient in Python for RAG tasks.
    *   **Trade-off:** Python's concurrency is weaker than C#, but for an I/O bound task (waiting on Notion API), Python's `asyncio` is sufficient.

### **Local LLM Backend: Ollama**
*   **Verdict:** **Ollama**
*   **Justification:**
    *   **VRAM Efficiency:** Ollama is highly optimized for running GGUF (quantized) models. It manages the model layers between GPU and CPU automatically.
    *   **Model Selection:** **Llama 3.1 8B (Quantized to 4-bit or 5-bit)**.
        *   Size: ~5-6 GB VRAM.
        *   Fits comfortably within your 8GB VRAM buffer, leaving ~2GB for the context window (KV Cache) and display output.

### **Vector Database: Qdrant (Local Mode) or Chroma**
*   **Verdict:** **Qdrant**
*   **Justification:**
    *   **Performance:** Written in Rust, extremely fast and memory-efficient.
    *   **Deployment:** Qdrant can run in a "local mode" (embedded) in Python without needing a separate Docker container, or as a very lightweight binary. This saves RAM compared to Java-based DBs or heavier solutions.
    *   **Filtering:** Excellent support for metadata filtering (crucial for filtering by Notion tags or dates).

### **Orchestration: LlamaIndex**
*   **Verdict:** **LlamaIndex** (over LangChain)
*   **Justification:**
    *   LlamaIndex is specialized for **Data Ingestion and Retrieval**. It treats data as a first-class citizen.
    *   It has superior strategies for "Updating" indexes (handling document updates) compared to LangChain's more general-purpose approach.

---

## 3. The "Syncing Strategy"

To avoid re-embedding the entire workspace (which costs time and compute), we need an **Incremental Sync Strategy**.

### **1. Change Detection (The "Delta" Scan)**
*   **Mechanism:** Use the Notion Search API to query pages.
*   **Filter:** Query for pages where `last_edited_time` > `local_last_sync_timestamp`.
*   **Storage:** Maintain a lightweight SQLite table or JSON file locally:
    ```json
    {
      "page_id": "xyz-123",
      "last_synced_hash": "abc-hash",
      "last_edited_time": "2023-10-27T10:00:00Z"
    }
    ```

### **2. Handling Rate Limits**
*   Notion API allows an average of 3 requests per second.
*   **Strategy:** Implement a "Token Bucket" rate limiter or simple exponential backoff in the ingestion loop.
*   **Library:** `tenacity` (Python library) is excellent for decorating API calls with retry logic.

### **3. The Pipeline**
1.  **Poll:** Every X minutes (configurable), query Notion for changed pages.
2.  **Diff:** Compare returned pages against the local state.
3.  **Delete:** If a page is archived/deleted in Notion, remove its vectors from Qdrant.
4.  **Upsert:** If modified, re-fetch content -> Chunk -> Embed -> Upsert to Qdrant (overwriting old vectors).

---

## 4. Performance Expectations

### **Hardware Reality Check**
*   **System:** i5-14400F, RTX 4060 (8GB), 16GB RAM.

### **Token Generation Speed**
*   **Model:** Llama 3.1 8B (4-bit Quantization).
*   **Estimated Speed:** **40 - 60 tokens per second**.
*   **Experience:** This is faster than reading speed. It will feel very snappy.

### **RAM Consumption Analysis (Worst Case)**
| Component | Estimated Usage | Notes |
| :--- | :--- | :--- |
| **Windows 11 OS** | 4.0 GB | Baseline |
| **VS Code + Extensions** | 1.5 GB | Electron apps are heavy |
| **Browser (Research)** | 2.0 GB | 10-15 tabs |
| **Ollama (Model Weights)** | 0.5 GB | Most weights are in VRAM (GPU) |
| **Ollama (Context/Overhead)**| 1.0 GB | System RAM buffer |
| **Python App (LlamaIndex)** | 0.8 GB | Loading embeddings/libraries |
| **Qdrant (Vector Index)** | 0.5 GB | Depends on # of Notion pages |
| **TOTAL** | **~10.3 GB** | **SAFE** (within 16GB limit) |

*Warning:* If you run Docker Desktop, add +2GB overhead. Prefer running tools natively (Python venv, Ollama Windows executable) to save RAM.

---

## 5. Project Foundation Questionnaire

Before writing code, we must scope the MVP to ensure success.

### **Data Structure & Chunking**
1.  **Hierarchy:** Notion pages are trees of blocks. Do we want to preserve the parent-child relationship in the vector store? (e.g., If a paragraph is inside a "Toggle" block, should the toggle title be included in the chunk context?)
2.  **Noise:** How do we handle "Linked Databases" or "Table Views" in Notion? These often contain high-density data that confuses semantic search if not formatted as text.

### **User Interface**
1.  **Interaction Mode:**
    *   *Option A:* **CLI (Command Line)** - Easiest, zero RAM overhead.
    *   *Option B:* **Streamlit/Gradio** - Web UI, runs locally in browser. Good visualization, low dev effort.
    *   *Option C:* **Desktop App (Electron/Tauri)** - High RAM usage. **Avoid for this hardware.**
    *   *Recommendation:* **Streamlit**. It looks professional but is just a Python script.

### **Scope**
1.  **Images:** Do we need to OCR images inside Notion pages? (Adds significant complexity and dependency overhead).
2.  **History:** Does the chat need to remember previous questions in the conversation (Multi-turn), or is every question isolated?

---

## Next Steps
1.  **Confirm Stack:** Approve Python + LlamaIndex + Qdrant + Ollama.
2.  **API Setup:** Create a Notion Integration token.
3.  **Scaffold:** Initialize the Python project structure.
