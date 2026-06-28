# VectorDB -  Building a Vector Database from Scratch in C++

Ever wondered what's actually happening inside AI when you do a semantic search? I did too, so I built one from scratch to find out.

This is a fully working **Vector Database** written in C++, with a web UI you can actually play around with. It runs three different search algorithms side by side so you can compare them, and there's a full **RAG pipeline** hooked up to a local LLM via Ollama - no API keys, no cloud, everything runs on your laptop.

> This started as an educational project, but it turned into something I'm genuinely proud of. If you've ever been curious about how the internals of a vector DB work, this is a good place to start.

---

## What's inside

| Feature | What it does |
|---|---|
| **3 Search Algorithms** | HNSW (the production-grade one), KD-Tree, and plain Brute Force - you can run all three and watch the speed difference |
| **3 Distance Metrics** | Cosine similarity, Euclidean distance, Manhattan distance |
| **16D Demo Vectors** | 20 pre-loaded vectors across 4 categories (CS, Math, Food, Sports) so you can jump in immediately |
| **2D PCA Scatter Plot** | Live visualization of the semantic space - you can actually *see* the clusters form |
| **Real Document Embedding** | Paste any text → Ollama converts it to a real 768D vector with `nomic-embed-text` |
| **RAG Pipeline** | Ask questions about your documents → HNSW finds the relevant chunks → a local LLM writes the answer |
| **Full REST API** | CRUD endpoints for insert, delete, search, benchmark, and HNSW graph inspection |

---

## How it all fits together

```
Your Text
    │
    ▼
Ollama (nomic-embed-text)          ← turns your text into a 768-dimensional vector
    │
    ▼
HNSW Index (C++)                   ← slots that vector into a multilayer graph
    │
    ▼
Semantic Search                    ← finds the nearest neighbors in vector space
    │
    ▼
Ollama (llama3.2)                  ← reads the retrieved chunks and writes an answer
    │
    ▼
Answer
```

The star of the show is **HNSW (Hierarchical Navigable Small World)** - the same algorithm that powers Pinecone, Weaviate, Chroma, and Milvus. It builds a multilayer graph where each layer gets progressively sparser. Searches start at the top (the "highway") and zoom in layer by layer, which gets you O(log N) performance instead of the O(N) slog of brute force.

---

## What you'll need

Just three things:

1. **MSYS2** - gives you the g++ compiler on Windows
2. **Git** - for cloning the repo
3. **Ollama** - runs the AI models locally

---

## Setup (Windows, step by step)

### Step 1 - Get the C++ compiler (MSYS2)

1. Head to **https://www.msys2.org** and grab the installer
2. Run it, keep the default path (`C:\msys64`)
3. Open **MSYS2 UCRT64** from your Start Menu (it has an orange icon)
4. Run this to update the package manager:

```bash
pacman -Syu
```

If it asks you to close and reopen the terminal, do that, then run the install command:

```bash
pacman -S mingw-w64-ucrt-x86_64-gcc
```

5. Now add g++ to your Windows PATH so PowerShell can find it:
   - Press `Win + R`, type `sysdm.cpl`, hit Enter
   - Go to **Advanced** → **Environment Variables**
   - Find **Path** under System variables, click **Edit**
   - Add a new entry: `C:\msys64\ucrt64\bin`
   - Click OK all the way out
   - Open a **new** PowerShell and check it worked:
   ```
   g++ --version
   ```
   You should see something like `g++ (GCC) 15.x.x` - you're good.

---

### Step 2 - Install Git

Nothing fancy here:

1. Download Git for Windows from **https://git-scm.com/download/win**
2. Run the installer with the default options
3. Verify:
```
git --version
```

---

### Step 3 - Install Ollama (the local AI models)

1. Go to **https://ollama.com** and download the Windows version
2. Install it - Ollama will start automatically and sit in your system tray
3. Open PowerShell and pull the two models you'll need:

```powershell
ollama pull nomic-embed-text
```
*This is the embedding model (~274 MB). It converts text into vectors.*

```powershell
ollama pull llama3.2
```
*This is the language model (~2 GB). It reads the retrieved context and writes answers.*

4. Double-check both are there:
```powershell
ollama list
```

> **Heads up on specs:** 8GB RAM is recommended. Both models together use about 3GB, so give your laptop a moment on first load.

---

### Step 4 - Clone the repo

```powershell
git clone https://github.com/YOUR_USERNAME/VectorDB.git
cd VectorDB
```

*(Swap in the actual GitHub username)*

---

### Step 5 - Compile

Inside the `VectorDB` folder:

```powershell
g++ -std=c++17 -O2 main.cpp -o db -lws2_32
```

This spits out `db.exe`. Takes maybe 10–20 seconds.

> **If something goes wrong:**
> - `g++: command not found` → MSYS2 isn't in your PATH yet, go back to Step 1
> - `undefined reference to WSA...` → you're missing `-lws2_32` at the end of the command
> - Taking forever? Drop the `-O2` flag - it'll compile faster (but the binary runs slower)

---

### Step 6 - Start everything up

**Terminal 1** - Make sure Ollama is running:
```powershell
ollama serve
```
*(If it's already running in your system tray, skip this.)*

**Terminal 2** - Start the VectorDB server:
```powershell
./db
```

If everything's working, you'll see:
```
=== VectorDB Engine ===
http://localhost:8080
20 demo vectors | 16 dims | HNSW+KD-Tree+BruteForce
Ollama: ONLINE
  embed model: nomic-embed-text  gen model: llama3.2
```

Now open your browser and go to:
```
http://localhost:8080
```

---

## Using the app

### Tab 1: Search (the demo vectors)

This is the most fun tab to start with. Type in any concept - `binary tree`, `sushi`, `basketball`, `calculus` - pick an algorithm and a distance metric, and hit **⚡ SEARCH**. Results show up with distances, and the matching point lights up on the scatter plot.

Hit **▶ COMPARE ALL ALGOS** to run all three algorithms back-to-back and see the actual timing difference.

The scatter plot shows all 20 vectors projected down to 2D using PCA. Notice how the four categories (CS, Math, Food, Sports) naturally cluster together - that's semantic similarity made visible.

### Tab 2: Documents (real embeddings)

This is where it gets real. Ollama generates actual **768-dimensional embeddings** from whatever text you paste in.

1. Give it a title (e.g., `Operating Systems Notes`)
2. Paste in your text - lecture notes, textbook paragraphs, anything
3. Click **⚡ EMBED & INSERT**

Long documents get automatically split into overlapping 250-word chunks. Each chunk gets its own embedding and lives in a separate HNSW index.

### Tab 3: Ask AI (the RAG pipeline)

Insert some documents in Tab 2 first, then come here and ask questions about them.

Behind the scenes, here's what happens:
```
1. Your question → nomic-embed-text turns it into a 768D vector
2. HNSW search → finds the 3 most semantically similar chunks
3. Those chunks → sent as context to llama3.2
4. llama3.2 → writes an answer grounded in your documents
```

The answer streams in with a typewriter effect. Click the **context chips** below it to see exactly which chunks the AI was working from.

---

## REST API

The server exposes a full REST API at `http://localhost:8080` if you want to poke at it directly.

### Demo vector endpoints

| Method | Endpoint | What it does |
|---|---|---|
| `GET` | `/search?v=f1,f2,...&k=5&metric=cosine&algo=hnsw` | K-NN search |
| `POST` | `/insert` | Insert a new demo vector |
| `DELETE` | `/delete/:id` | Remove by ID |
| `GET` | `/items` | List everything |
| `GET` | `/benchmark?v=...&k=5&metric=cosine` | Run all 3 algorithms and compare |
| `GET` | `/hnsw-info` | Peek at the HNSW graph structure |
| `GET` | `/stats` | General database stats |

### Document & RAG endpoints

| Method | Endpoint | Body | What it does |
|---|---|---|---|
| `POST` | `/doc/insert` | `{"title":"...","text":"..."}` | Embed and store a document |
| `GET` | `/doc/list` | - | List all stored documents |
| `DELETE` | `/doc/delete/:id` | - | Remove a document chunk |
| `POST` | `/doc/ask` | `{"question":"...","k":3}` | RAG: retrieve + generate answer |
| `GET` | `/status` | - | Ollama connection and model info |

### Quick examples

```powershell
# Search
curl "http://localhost:8080/search?v=0.9,0.8,0.7,0.6,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1&k=3&metric=cosine&algo=hnsw"

# Ask a question
curl -X POST http://localhost:8080/doc/ask `
  -H "Content-Type: application/json" `
  -d '{"question":"What is dynamic programming?","k":3}'
```

---

## Project structure

```
VectorDB/
├── main.cpp        ← C++ backend (HNSW, KD-Tree, BruteForce, REST API, RAG)
├── httplib.h       ← Single-header HTTP server library (cpp-httplib)
├── index.html      ← Frontend (scatter plot, chat UI, benchmark view)
└── README.md       ← You are here
```

### Architecture at a glance

```
BruteForce          O(N·d)      Exact, slow, the baseline
KDTree              O(log N)    Exact, fast in low dimensions
HNSW                O(log N)    Approximate, fast in high dimensions

VectorDB            Unified interface over all 3 (for the 16D demo vectors)
DocumentDB          HNSW-only index for real Ollama embeddings (768D)
OllamaClient        HTTP client talking to /api/embeddings + /api/generate
```

---

## A bit more on the algorithms

### HNSW - why it's the one that matters

When you insert a node, it randomly gets assigned to a max layer. Layer 0 has every node, densely connected; higher layers have exponentially fewer nodes with longer-range connections (think: highway exits vs local streets).

**Insert:** Greedy descent from the top layer, beam search at each layer from the assigned max down to 0, connect to the M nearest neighbors bidirectionally.

**Search:** Same greedy descent, but at layer 0 you expand to `ef` nearest candidates using a priority queue.

**Why it's fast:** The upper layers get you to the right neighborhood quickly. Layer 0 closes in on the answer. You never scan everything.

### KD-Tree - fast until it isn't

Binary space partitioning along alternating dimensions. The clever bit: it prunes entire subtrees when the closest possible point in that subtree can't beat your current best.

The catch: this pruning falls apart in high dimensions. Once you're past ~20 dimensions, almost nothing gets pruned and you're basically doing brute force anyway. At 768D, KD-Tree and brute force perform almost identically.

### Why HNSW still wins at 768D

KD-Tree's pruning relies on axis-aligned distance bounds. In high-dimensional space, almost everything lives near the surface of the hypersphere, so those bounds never help. HNSW's graph structure sidesteps this entirely - it doesn't care how many dimensions you have.

---

## Troubleshooting

| Problem | What to try |
|---|---|
| `Ollama: OFFLINE` in the header | Run `ollama serve` in a terminal |
| Embedding is taking forever | Ollama is probably downloading the model for the first time - give it a couple minutes |
| `g++: command not found` | Add `C:\msys64\ucrt64\bin` to your Windows PATH (Step 1) |
| Port 8080 already in use | Find the process: `netstat -ano \| findstr 8080`, then kill it: `taskkill /PID <pid> /F` |
| LLM responses are really slow | Totally normal on a laptop CPU - llama3.2 takes 10–30s. See below for a faster option. |

### Switching to a smaller model

If llama3.2 is grinding your laptop to a halt, try the 1B version - it's much faster:

```powershell
ollama pull llama3.2:1b
```

Then in `main.cpp`, find where `genModel` is set and change it:
```cpp
std::string genModel = "llama3.2:1b";
```

Recompile and restart. Noticeably snappier.
