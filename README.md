# ArcLens

**AI-powered architecture diagram generator for ArcGIS Enterprise on Kubernetes**

## The Problem

ArcGIS Enterprise on Kubernetes runs 20+ microservices — Portal, Server, Feature Server, Map Server, multiple data stores, ingress controllers, queues, and more. Understanding how they connect requires reading through dozens of documentation pages. There is no single auto-generated visual that shows the full picture.

New engineers spend days building a mental model. Support engineers manually trace component relationships when troubleshooting. Customers deploying on EKS, AKS, or OpenShift struggle to see what their architecture actually looks like.

## The Solution

Paste a URL. Get an architecture diagram. That's it.

```
URL → Extract Content → Summarize Structure → Generate Code → Render Diagram → PNG
```

ArcLens chains two AI agents with a self-correcting execution loop:

1. **Summarizer Agent** — Reads raw page content and extracts a structured JSON with components, relationships, data flows, deployment context, and configuration parameters. Uses domain-specific knowledge of AEOK component names and architecture layers.

2. **Diagram Agent** — Takes the structured JSON and generates executable Python code using the `diagrams` library. Maps each AEOK component to the correct Kubernetes icon (Deploy, StatefulSet, Ingress, PVC, etc.) with cloud-specific icons for AWS, Azure, and on-prem infrastructure.

3. **Self-Correcting Execution** — Generated code runs in an isolated subprocess. If it fails, the error is fed back to the AI for automatic correction. Common mistakes are pre-patched before execution. Up to 3 retry attempts.

---

## Quick Start

### Prerequisites

- Python 3.12
- Graphviz (`brew install graphviz` on macOS)
- OpenAI API key
- Tavily API key ([tavily.com](https://tavily.com))

### Setup

```bash
git clone https://github.com/yourusername/arclens.git
cd arclens

python3.12 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# Edit .env and add your OPENAI_API_KEY and TAVILY_API_KEY
```

### Run

```bash
# Process all 6 default AEOK documentation URLs
python tavily/tavily.py

# Process a single URL
python tavily/tavily.py --urls "https://enterprise-k8s.arcgis.com/en/11.4/introduction/what-is-arcgis-enterprise-kubernetes.htm"

# Process multiple specific URLs
python tavily/tavily.py --urls "https://url1.com" "https://url2.com" "https://url3.com"
```

Output PNGs are saved to `tavily/output/`.

---

## How It Works

### Pipeline Flow

```
┌─────────────────────────────────────────────────────┐
│                    USER INPUT                        │
│         One or more documentation URLs               │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              TAVILY EXTRACT                          │
│  Pulls full page content as clean markdown           │
│  Handles JS-rendered pages, strips navigation        │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│           SUMMARIZER AGENT (LLM)                     │
│                                                      │
│  Input:  Raw markdown content (up to 15K tokens)     │
│  Output: Structured JSON                             │
│                                                      │
│  Extracts:                                           │
│  ├── Components (name, type, layer, runtime)         │
│  ├── Relationships (from, to, type)                  │
│  ├── Data Flows (name, ordered steps)                │
│  ├── Deployment Context (platforms, storage, etc.)   │
│  ├── Configuration Parameters                        │
│  └── Document Metadata (title, doc_type, version)    │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│          DIAGRAM AGENT (LLM)                         │
│                                                      │
│  Input:  Structured JSON from Summarizer             │
│  Output: Executable Python code                      │
│                                                      │
│  Features:                                           │
│  ├── 30+ AEOK component-to-icon mappings             │
│  ├── Cluster grouping by architecture tier           │
│  ├── Color-coded edges by relationship type          │
│  ├── Adapts diagram style based on doc_type          │
│  └── Supports K8s, AWS, Azure, on-prem icons        │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│       SELF-CORRECTING EXECUTION LOOP                 │
│                                                      │
│  1. Write generated code to temp file                │
│  2. Execute in isolated subprocess (120s timeout)    │
│  3. If success → return PNG path                     │
│  4. If failure → feed error back to LLM              │
│  5. LLM generates corrected code                     │
│  6. Repeat up to 3 attempts                          │
│                                                      │
│  Pre-execution patches:                              │
│  └── Fixes Cluster(style=...) → graph_attr={...}    │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│                  OUTPUT                              │
│         PNG architecture diagram per URL             │
│         Saved to tavily/output/                      │
└─────────────────────────────────────────────────────┘
```

### Diagram Adaptation by Document Type

The system generates different diagram styles based on what it's reading:

| Document Type | Diagram Style |
|---------------|---------------|
| Architecture Overview | Full architecture with all components, clusters, and connections |
| Deployment Guide | Infrastructure topology showing platforms and deployment flow |
| Troubleshooting | Affected components highlighted with red edges |
| Tutorial | Data flow diagram showing workflow steps |
| Release Notes | Changed components highlighted (new/deprecated) |
| Configuration | Detail view focused on the configured component |

---

## Project Structure

```
arclens/
├── .env.example              # Environment variables template
├── requirements.txt          # Python dependencies
├── llm_factory.py            # LLM abstraction (OpenAI / Anthropic)
└── tavily/
    ├── tavily.py             # Main pipeline — extract, summarize, generate, execute
    ├── summaizer.yaml        # Summarizer Agent system prompt
    ├── architecture.yaml     # Diagram Agent system prompt (with icon inventory)
    └── output/               # Generated PNG diagrams
```

---

## Supported Content Sources

ArcLens has been tested with:

- **Esri Official Docs** — enterprise-k8s.arcgis.com (deployment guides, architecture overviews, tutorials)
- **Esri Blog** — esri.com/arcgis-blog (release announcements, what's new posts)
- **Community Articles** — Medium, community.esri.com
- **Any URL** — Works with any web page that describes AEOK architecture or components

### Default URLs (built-in)

| URL | Content |
|-----|---------|
| Run the Deployment Script | Deployment flow, prerequisites, script parameters |
| End-to-End Deep Learning | GeoAI workflow through AEOK services |
| What is AEOK | Full architecture overview with all components |
| What's New in 12.0 | Release notes, new features, changed components |
| Publish Web Tools | Publishing workflow through Server and data stores |
| Under the Hood (Medium) | Internal architecture deep dive |

---

## AEOK Domain Knowledge

The system understands ArcGIS Enterprise on Kubernetes at the component level:

**Services Tier** — Portal, ArcGIS Server, Map Server, Feature Server, Geocode, Sharing, Catalog, Apps, Help

**Management & Tools** — Admin, Queue, Indexer, Metrics, System Tools, Utility Tools, Logs, Manager

**Data Stores** — Relational (PostgreSQL), Object Store (MinIO), Tile Cache, Spatio Temporal (Elasticsearch)

**Infrastructure** — Ingress Controller (NGINX), Load Balancer, Container Registry, Persistent Volumes

**Cloud Platforms** — Amazon EKS, Azure AKS, OpenShift, on-premises Kubernetes

**Version Awareness** — Understands differences between 11.x and 12.x (e.g., Admin UI datastore config support added in 12.0)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3.12 |
| LLM Framework | LangChain 0.3.x |
| LLM | OpenAI GPT-4o-mini (swappable) |
| Content Extraction | Tavily Extract API |
| Diagram Rendering | mingrammer/diagrams + Graphviz |
| LLM Abstraction | Custom factory supporting OpenAI and Anthropic |
| Execution | Isolated subprocess with timeout and retry |

---

## Future Roadmap

- **Log Diagnosis Pipeline** — Paste AEOK pod logs → get root cause analysis with failure diagram highlighting the affected component path
- **Deployment Audit Pipeline** — Paste kubectl output or Helm values → get a gap analysis comparing your deployment against Esri's reference architecture
- **Streamlit UI** — Web interface with drag-and-drop URL input, real-time progress, and interactive diagram viewer
- **Parallel Processing** — Concurrent URL processing using ThreadPoolExecutor for batch speedup
- **Summary Caching** — Cache extracted summaries by content hash to skip LLM calls on repeat URLs

---

## License

MIT
