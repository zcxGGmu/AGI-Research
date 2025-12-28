# AGI-Research

A compilation of research reports and technical analysis on AGI (Artificial General Intelligence) related projects.

## Overview

This repository contains in-depth technical analysis reports on cutting-edge AGI infrastructure projects, with a focus on AI operating systems, agent frameworks, and supporting technologies.

## Project Structure

```
AGI-Research/
├── aios/                    # AIOS (AI Agent Operating System) analysis
│   ├── codex/              # AIOS technical analysis (Codex perspective)
│   └── cc/                 # AIOS deep technical analysis
├── trading/                 # Trading-related AI research (placeholder)
└── README.md               # This file
```

## Research Reports

### AIOS (AI Agent Operating System)

AIOS is an operating system kernel designed for AI Agents, adopting Unix-like design principles to provide system call abstractions, resource scheduling, memory management, storage management, and tool management.

**Key Features:**
- System call abstraction layer for unified agent-kernel communication
- Multi-LLM backend support (OpenAI, Anthropic, Google, local models, etc.)
- Vector-based memory management with semantic search
- Semantic file system (LSFS) with AI-powered content indexing
- React-style hooks for state management
- Multiple scheduling strategies (FIFO, Round-Robin)
- Virtual environment support (Docker, AWS, Azure, GCP)

**Reports:**
- [`aios/codex/AIOS_Analysis_Report.md`](aios/codex/AIOS_Analysis_Report.md) - Technical analysis overview
- [`aios/cc/AIOS_Analysis_Report.md`](aios/cc/AIOS_Analysis_Report.md) - Comprehensive deep-dive analysis including:
  - Complete directory structure
  - Core architecture design patterns
  - Module-by-module breakdown
  - Workflow and data flow analysis
  - Deployment modes and API interfaces

## Technology Stack Covered

| Category | Technologies |
|----------|--------------|
| **LLM Integration** | LiteLLM, OpenAI, Anthropic, Google Gemini, Ollama, vLLM, SGLang |
| **Vector Databases** | ChromaDB, Qdrant |
| **Web Frameworks** | FastAPI, Uvicorn |
| **Storage** | Redis, LlamaIndex |
| **Programming Languages** | Python 3.11+, Rust (experimental) |

## Purpose

This repository serves as:
1. **Learning Resource**: Detailed technical analysis for understanding AGI infrastructure
2. **Reference Documentation**: Architecture patterns and design decisions for building AI systems
3. **Research Notes**: Ongoing research into cutting-edge AI technologies

## License

This research repository is for educational and reference purposes. Please refer to individual project repositories for their specific licenses.

## Contributing

This is a personal research repository. For suggestions or corrections, please refer to the original projects being analyzed.
