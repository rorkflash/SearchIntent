# ⚡ SearchIntent

> **A zero-cost, zero-hallucination NLP bridge for hybrid search architectures.**

SearchIntent is a lightweight backend microservice that acts as an intelligent bridge between a user's natural language query and your vector database. Instead of relying on slow, expensive, and hallucination-prone generative LLMs (like OpenAI or Claude), SearchIntent uses **GLiNER** (Generalist and Lightweight Named Entity Recognition) to execute zero-shot intent parsing completely locally on your CPU.

**[🌐 See SearchIntent live in production at MovieTrip.me](https://movietrip.me)**

---

## 💡 The Problem
Modern search is broken:
- **Keyword Search** misses the "vibe" of what users want.
- **Pure Vector/Semantic Search** fails at strict hard filters (e.g., returning a movie filmed in Chicago when the user strictly asked for "New York").
- **LLM Intent Parsing** is too expensive at scale, too slow for search bars, and prone to hallucinations.

## 🚀 The Solution
SearchIntent instantly isolates **hard metadata** (like locations, dates, or genres) from **abstract semantics** (the "vibe"). It packages this intent into a predictable JSON object that can be routed to *any* database (Supabase, Pinecone, Qdrant) to execute a flawless Hybrid Search.

### Key Features
* 💸 **Absolute $0 API Costs:** Runs completely locally on standard CPUs. You never pay per search query.
* 🛡️ **Zero Hallucinations:** Uses an *extractive* NLP model. It will never invent a fake location or guess an incorrect category.
* 🔌 **Database Agnostic:** Connects seamlessly to any Vector DB or traditional SQL/NoSQL database.
* ⚡ **Blazing Fast:** Designed to be the fastest step in your search pipeline, minimizing user-facing latency.

---

## 🛠️ Installation

SearchIntent is built on Python and utilizes the official `gliner` package.

```bash
# Create a virtual environment
python -m venv venv
source venv/bin/activate

# Install GLiNER and requirements
pip install gliner
