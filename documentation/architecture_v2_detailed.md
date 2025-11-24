# Local RAG "Second Brain" - Architecture V2 (Detailed)

**Date:** November 22, 2025
**Role:** Principal Software Architect & AI Systems Engineer
**Subject:** Refined Architectural Blueprint & Deep Dive into OS Constraints

---

## Executive Summary

This V2 document refines the initial architectural research based on specific user feedback. Key shifts include a focus on **Windows-native optimization**, a **Streamlit-based Web UI** (avoiding CLI), and a sophisticated **Memory Module** that allows the AI to retain context across conversations.

The core constraint remains the **16GB RAM** limit on a **Windows 11** machine. This document explicitly addresses why the OS choice dictates specific architectural decisions.

---

## 1. The Operating System Factor: Why Windows Matters

You asked why the OS is a critical variable in the equation. In high-performance AI engineering, the OS is not just a container; it is a resource consumer that competes with your application.

### **A. The "Windows Tax" on RAM**
*   **Baseline Consumption:** A fresh Windows 11 Pro installation typically consumes **3.5GB - 4.5GB** of RAM just to idle (running Defender, Telemetry, UI, Search Indexing).
*   **Comparison:** A minimal Linux server (headless) consumes ~400MB.
*   **Impact:** On a 16GB machine, you effectively only have **~11-12GB** of "free" memory for your application before hitting the swap file (disk), which destroys performance.

### **B. The WSL2 Dilemma (Windows Subsystem for Linux)**
Many AI tutorials recommend using WSL2. **For your specific hardware, I recommend AGAINST using WSL2.**
*   **The Problem:** WSL2 runs a lightweight Virtual Machine. It reserves a fixed chunk of RAM (often 50% or 8GB by default) for the Linux kernel.
*   **The Cost:** If you run your Vector DB and Python app inside WSL2, you are running *two* operating systems simultaneously. This overhead is affordable on 32GB+ systems, but on 16GB, it is a critical bottleneck.
*   **Recommendation:** **Run Native Windows Python.**
    *   Use PowerShell.
    *   Install Python for Windows.
    *   This allows the application to share memory directly with the host OS without the virtualization penalty.

### **C. GPU Driver Access (CUDA)**
*   Fortunately, NVIDIA's Windows drivers are excellent. Ollama on Windows handles CUDA offloading seamlessly.
*   **Constraint:** Windows reserves ~0.5GB - 1GB of VRAM for the desktop window manager (rendering your windows/animations). This reduces your 8GB card to effectively **~7GB** for the model.

---

## 2. Architectural Design: The "Memory-Augmented" Monolith

We retain the **Async Modular Monolith** pattern but introduce a specific **Memory Module** to handle your requirement for "building knowledge about itself."

### **Core Components**

1.  **Ingestion Engine (Background Service)**
    *   **Role:** Connects to Notion API.
    *   **Strategy:** "Flattener." Instead of preserving the strict tree hierarchy, we treat notes as "Knowledge Nodes."
    *   **Metadata Extraction:** Crucial for your "Links and Backlinks" requirement. Every chunk will carry metadata: `source_url`, `linked_page_ids`, `tags`, `created_time`.

2.  **The Brain (RAG Engine)**
    *   **Orchestrator:** LlamaIndex.
    *   **Retrieval:** Hybrid Search (Keyword + Semantic).
    *   **Synthesis:** Generates answers with explicit citations (links back to Notion).

3.  **Memory Module (New)**
    *   **Short-term Memory:** Stores the last N turns of the *current* conversation (Context Window).
    *   **Long-term Memory (Fact Store):** A separate collection in Qdrant.
    *   **Mechanism:** When you state a fact (e.g., "I prefer sci-fi books"), the AI extracts this as a triple `(User, prefers, Sci-Fi)` or a text snippet and saves it. Future queries check this "Fact Store" first.

4.  **User Interface (MVP)**
    *   **Tech:** **Streamlit**.
    *   **Why:** It runs in the browser (Chrome/Edge) which you already have open. It renders Markdown tables and links beautifully. It avoids the overhead of running a separate Electron app (which is essentially another Chrome browser instance).

---

## 3. Data Strategy: Handling Notion Content

### **A. Hierarchy vs. Linking**
You specified that strict hierarchy is less important than the content and links.
*   **Chunking Strategy:** We will use **"Document-Level Chunking"**.
    *   A Notion Page is the parent.
    *   We split the text into 512-token chunks.
    *   **Crucial:** We append the "Page Title" and "Backlinks" to the *text* of every chunk before embedding.
    *   *Example Chunk:* `[Page: Book Recommendations] [Linked to: Reading List 2024] ...text content of the book review...`
    *   This ensures that even if a chunk is isolated, the AI knows where it came from and what it connects to.

### **B. OCR & Images (Lightweight Approach)**
*   **Requirement:** Identify "what the image is about" without heavy OCR.
*   **Solution:** Notion API provides the `caption` and `name` of image blocks.
*   **Implementation:** We will index the **Image Caption** and **File Name**. We will *not* run a vision model over the pixel data in Phase 1. This saves massive compute resources.
    *   *Future Upgrade:* If you upgrade hardware, we can swap in Llama 3.2 Vision to describe images.

---

## 4. Technology Stack (Finalized for MVP)

| Component | Choice | Justification for 16GB/RTX4060 |
| :--- | :--- | :--- |
| **Language** | **Python 3.11+** | Native Windows install. Best ecosystem. |
| **LLM Host** | **Ollama (Windows)** | Efficient VRAM management. |
| **Model** | **Llama 3.1 8B (Q4_K_M)** | Fits in ~5.5GB VRAM. Smart enough for summarization. |
| **Vector DB** | **Qdrant (Embedded)** | Runs *inside* the Python process. Zero network overhead. |
| **Framework** | **LlamaIndex** | Best for "Chat Engine" with memory and data connectors. |
| **UI** | **Streamlit** | Lightweight web interface. |

---

## 5. Sync & Memory Strategy

### **A. The "Fact Extraction" Pipeline**
To satisfy: *"AI could also capture some important facts and save a memory of sorts."*

1.  **User Input:** "I'm trying to learn Rust programming this month."
2.  **Parallel Process:**
    *   *Main Thread:* Searches Notion for "Rust", generates answer.
    *   *Background Thread:* Analyzes input. Detects "User Goal".
3.  **Storage:** Saves a "Memory" object in Qdrant:
    *   `Content`: "User is learning Rust in Nov 2025."
    *   `Type`: "UserFact"
4.  **Recall:** Next time you ask "What should I study?", the AI retrieves this fact and prioritizes Rust resources.

### **B. Incremental Sync**
*   **Frequency:** Every 10 minutes (Background thread).
*   **Logic:**
    1.  Fetch `search` from Notion filtered by `last_edited_time`.
    2.  If page modified: Delete old chunks -> Re-chunk -> Re-embed.
    3.  **Rate Limit Safety:** Use `tenacity` library to wait if Notion sends 429 (Too Many Requests).

---

## 6. Next Steps

1.  **Environment Setup:**
    *   Install Python 3.11 (Windows Installer).
    *   Install Ollama for Windows.
    *   `pip install llama-index qdrant-client streamlit notion-client`
2.  **Prototype Phase 1:**
    *   Build the `NotionIngestor` class to fetch 5 pages and print their "Chunks" to see if the metadata strategy works.

