# AI Notes

Exploring ideas, prompts, and personal insights from curious conversations with AI. This repository contains a collection of in-depth technical guides and prompt engineering templates generated from these explorations.

-----

## 🗺️ Learning Paths

*   [The Rust-Powered Architect: A Modern Path from Backend to AI](./learning_paths/the-rust-powered-architect.md)
    *   *Core Concepts: Structured Learning Path, Feynman Technique, Full-Stack Architecture, Backend (Rust, Postgres), Frontend (React), Infrastructure (Kafka, Observability), AI Engineering (RAG, Agents).*

-----

## 🚀 In-Depth Technical Guides

All guides are structured using the Feynman learning method to build deep, intuitive understanding of complex systems. 

### AI

*   [Building a Performant AI Agent Framework in Rust from Scratch](./technical_guides/ai/rust-ai-agent-framework-from-scratch.md)
    *   *Core Concepts: ReAct Paradigm, Modular Design (Traits), Tool Use, State Management, Async Orchestration.*
*   [Kubrick Course Learning Guide to Multimodal AI Systems](./technical_guides/ai/kubrick-course-learning-guide.md)
    *   *Core Concepts: Multimodal Agents, Model Context Protocol (MCP), Four Pillars (Pixeltable, FastMCP, Groq, Opik), Multi-Agent Collaboration, Observability.*
*   [LLM Reverse-Engineering: A Generalized Framework and Analysis of Modern Architectures](./technical_guides/ai/llm-reverse-engineering.md)
    *   *Core Concepts: Reverse-Engineering Framework, "Visualize, Hypothesize, Verify", Comparative Triage, Netron, Architectural Analysis (Gemma, DeepSeek, Llama, Qwen).*
*   [Mastering PyTorch for Large Language Models: From Fundamentals to Frontier](./technical_guides/ai/mastering-pytorch-for-llms.md)
    *   *Core Concepts: Tensors, Autograd, nn.Module, optim, Dataset, DataLoader.*
*   [The AURORA Project: A Production-Focused Guide to Building Multimodal AI Agents in Rust](./technical_guides/ai/rust-multimodel-ai-agents.md)
    *   *Core Concepts: Rust for AI, ReAct Loop, Model Context Protocol (MCP), Four Pillars (Axum, Qdrant, Tokio, Tracing), Fearless Concurrency.*
*   [The Engineer's Guide to Production-Ready Generative AI](./technical_guides/ai/the-engineer-guide-to-production-ready-generative-ai.md)
    *   *Core Concepts: AI Engineering, Transformers, Attention Mechanism, Prompt Engineering, RAG (Retrieval-Augmented Generation), AI Agents, LLMOps, LLM-as-a-Judge.*

### CSS

 *   [The Complete Guide to Mastering Tailwind CSS: From Foundations to v4 Production Mastery](./technical_guides/css/the-complete-guide-to-mastering-tailwind-css.md)     * *Core Concepts: Utility-First Philosophy, JIT Compilation, Oxide Engine (v4), CSS-First Configuration (`@theme`), Semantic Tokens, Container Queries.*

### Kafka

*   [Technical Deep Dive: Kafka Consumer Fault Tolerance and Rebalancing](./technical_guides/kafka/kafka-consumer-fault-tolerance-and-rebalancing.md)
    *   *Core Concepts: Fault Tolerance, At-Least-Once Delivery, Rebalancing, Idempotency, Dead Letter Queues (DLQ).*

### Observability

*   [A Comprehensive Guide to Production-Grade Monitoring with Grafana, Prometheus, and Loki](./technical_guides/observability/a-mcoprehensive-guide-to-production-grade-monitoring-with-grafana-prometheus-and-loki.md)
    *   *Core Concepts: PLG Stack, Shared Label Philosophy, Metrics & Logs Correlation, Docker Compose, Production Readiness (Security, HA, Performance), Alerting Philosophy, OpenTelemetry.*
      

### PostgreSQL

*   [PostgreSQL 17: A Feynman-Method Guide from Fundamentals to Production](./technical_guides/postgresql/gpostresql-a-feynman-method-guide-from-fundamentals-to-production.md)
    *   *Core Concepts: Architecture (MVCC, WAL), SQL Mastery, Performance Engineering (`EXPLAIN`, Indexing), Row-Level Security (RLS).*

### React

 *   [The Modern React Stack: A Feynman Guide from First Principles to Production](./technical_guides/react/the-modern-react-stack-a-feynman-guide-from-first-principles-to-production.md)     
     *   *Core Concepts: TanStack Router, TanStack Query, Zustand, Zod, Type-Safe Patterns.*

### Rust

*   [A Deep Dive into Rust: From Fundamentals to Production Patterns](./technical_guides/rust/a-edep-dive-into-rust-from-fundamentals-to-production-patterns.md)
    *   *Core Concepts: Structs and Enums, Traits, Concurrency with Tokio and Rayon, Design Patterns.*

*   [Architecting a State-Aware Conversational Bot in Rust: A Deep Dive](./technical_guides/rust/architecting-a-state-aware-conversational-bot.md)
    *   *Core Concepts: Axum Webhooks, Teloxide FSM, SQLx for Persistent State, Modular Monolith Architecture.*

*   [Type-Safe Full-Stack Development: Bridging Rust and TypeScript with Schemas](./technical_guides/rust/type-safe-full-stack-development-rust-typescript.md)
    *   *Core Concepts: SSoT, JSON Schema, `schemars`, `serde`, Enum Strategies, Generics, Automation, Ecosystem Benefits.*
-----

## 📚 Content Summaries

### Personal Finance
*   [Financial Literacy for Dummies (Like Me) with JL Collins](./content_summaries/personal_finance/financial-literacy-with-jl-collins.md)
    *   *Core Concepts: The Simple Path to Wealth, Spend Less Than You Earn, Index Fund Investing (VTSAX), Avoiding Debt, Financial Independence (FI), F.U. Money, Market Crashes.*

-----

## 🛠️ Prompt Engineering Templates

*   **[Technical Feature Request Document](./prompts/technical-feature-request-document.md)**: Generates a structured technical feature request document from a discussion for engineering teams.
*   **[Comprehensive Learning](./prompts/comprehensive-learning.md)**: Generates a structured and comprehensive learning plan for mastering a new topic, from fundamentals to advanced applications.
*   **[Learning Path Generator](./prompts/learning-path-generator.md)**: Creates a structured learning path article using the Feynman Teaching Technique to explain content in simple terms, with a mermaid chart and links to specific articles.
*   **[Feynman Article Generator](./prompts/feynman-article-generator.md)**: Generates a comprehensive technical article from a discussion using a structured, Feynman-inspired format.
*   **[Feynman Article Enhancer - Auto Suggestions](./prompts/feynman-article-enhancer.md)**: Enhances an existing educational article by elaborating on concepts, adding diagrams, and providing deeper examples, all guided by the Feynman method.
*   **[Feynman Article Refiner - Manual Suggestions](./prompts/feynman-article-refiner.md)**: Refines a technical article by systematically applying a specific list of improvement suggestions, ideal for iterative content development.
*   **[Video Transcription Enhancer](./prompts/video-transcription-enhancer.md)**: Reformats raw text transcripts for clarity, structure, and readability while preserving the original content verbatim.
*   **[Audio Transcript Refiner](./prompts/audio-transcript-refiner.md)**: Corrects errors in raw, machine-generated transcripts, improving grammar and structure to produce a clean and accurate text that preserves the speaker's original intent.
*   **[Audio Transcript Summarizer](./prompts/audio-transcript-summarizer.md)**: Analyzes raw, machine-generated transcripts to extract key information-such as decisions, action items, and conclusions-and presents it as a concise, time-stamped summary.
