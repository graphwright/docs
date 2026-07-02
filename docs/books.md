# Graphwright Publications

Three books covering the full pipeline from raw text to a language model that can reason over what that text contained.

---

## The Identity Server: Canonical Identity for Knowledge Graphs

The trustworthiness book. Full architecture of the identity server: the domain-agnostic base, the plugin contract, Docker deployment, caching, entity lifecycle, and the epistemic commons — the shared identifier infrastructure (MeSH, HGNC, RxNorm, UniProt) that makes graphs composable across sources.

---

## Knowledge Graphs from Unstructured Text

The extraction book. How to build a knowledge graph from raw documents using LLMs: schema design, the ingestion pipeline, identity resolution, provenance, and diagnostics. Includes the medlit biomedical reference implementation.

---

## BFS-QL: A Graph Query Protocol for Language Models

The interface book. How to serve a knowledge graph to a language model via a minimal, LLM-friendly protocol. Five MCP tools, a flat query format, backends for SPARQL, Postgres/pgvector, and Neo4j, and a worked implementation against the medlit graph. See the [BFS-QL Protocol](BFS-Query-Language.md) reference doc for the current spec.

---

Read in order, the three books cover: ensuring knowledge is **trustworthy** → getting knowledge **in** → getting it **out**.
