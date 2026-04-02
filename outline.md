# Documentation Outline -- Map of the Territory

## What This Is

A framework for building knowledge graphs over complex professional or academic
literature -- medical papers, legal documents, technical specifications. The core
belief: knowledge graphs capture essential structure in human reasoning and
expertise, and they provide a foundation for computers to reason in those same
domains.

A central concern throughout is **provenance**: every assertion in the graph
must trace back to a source document and location. Transparency and
traceability are not afterthoughts -- they are load-bearing.

---

## Map of Topics

### Concepts and Architecture

- `overview.md` -- Why knowledge graphs, core concepts, two-pass ingestion model
- `architecture.md` -- Component breakdown, module structure, immutability design

### Identity and Entity Resolution

- `canonical-ids-and-entity-resolution.md`
- The importance of "canonical" IDs -- adherence to accepted ontologies -- placing
  entities in the edifice of human knowledge -- reasoning across domains and across
  different source materials
- Entity lifecycle (provisional → canonical), authority lookup (UMLS, MeSH, etc.), synonym caching
- *(needed)* Deduplication -- how near-duplicate entities are merged or flagged

### Trust and Provenance

This is the philosophical core of the project and currently lives in fragments.
Needs a dedicated section:

- Source attribution -- which paper, which section or chunk, which claim
- Confidence scores -- what they mean and how they're set
- Conflicting claims -- how disagreements between sources are represented,
  not silently resolved
- Audit trail -- raw document → extracted entity → graph assertion

### Ingestion Pipeline

- `pipeline.md`
- Stages and interfaces (parser → extractor → resolver → embedder)
- *(needed)* Chunking strategies -- how documents get segmented before LLM processing
- *(needed)* Error handling -- partial extraction, retries, fallback behavior

### LLM Integration

- `deployment-and-operations.md` (partial)
- MCP server configuration
- *(needed)* MCP server in depth -- what tools/resources are exposed, usage patterns
- *(needed)* MCP troubleshooting -- what to do when surprises pop up
- *(needed)* LLM querying -- how LLMs navigate and query the graph
- *(needed)* Embeddings -- strategy for semantic search, how embeddings are stored/used
- *(needed)* Prompt design for extraction -- pointing to adapting-to-your-domain

### Schema Design

- `schema-design-guide.md` -- Defining DomainSchema, entity/relationship subclasses
- `adapting-to-your-domain.md` -- Step-by-step: define schema, write prompts,
  configure pipeline, validate

### Storage, Export, and Serving

- `storage-and-export.md`
- Bundle format (manifest.json + JSONL), export, query interfaces (REST, GraphQL, MCP)
- `deployment-and-operations.md` -- SQLite (dev) vs PostgreSQL (prod), Docker,
  Chainlit chat, scaling notes
- *(needed)* AWS deployment path -- target production architecture

### Examples

- `examples/medlit.md` -- Reference biomedical implementation (JATS XML, UMLS)
- `examples/sherlock.md` -- Simplified literary example, no external authorities

---

## Proposed Doc Structure

```
index.md                          ← map of territory, links to everything
overview.md                       ← why knowledge graphs, core concepts
architecture.md                   ← components, modules, data flow

identity/
  identity-server.md              ← identity server manages canonical ID lookup,
                                    maintenance, and caching
  canonical-ids.md                ← entity lifecycle, authority lookup
  deduplication.md                ← (new or split from canonical-ids)

trust/                            ← (new section)
  provenance.md                   ← source attribution, traceability
  conflicting-claims.md           ← (new)

ingestion/
  pipeline.md                     ← stages and interfaces
  chunking.md                     ← (new) document segmentation
  error-handling.md               ← (new) partial extraction, retries

llm-integration/
  mcp-server.md                   ← (expand from deployment)
  querying.md                     ← (new) how LLMs use the graph
  embeddings.md                   ← (new or expand from pipeline)
  extraction-prompts.md           ← (new or fold into adapting)

schema/
  schema-design-guide.md
  adapting-to-your-domain.md

storage-and-export.md
deployment-and-operations.md

examples/
  medlit.md
  sherlock.md
```

---

## Coding Standards and Practices

### Python

- Use `uv` for virtualenv management and running commands (`uv run`, `uv sync`,
  `uv pip`). Never invoke `python3` directly.
- Target Python 3.12 or 3.13.
- Use **pydantic models** for structured data. Prefer immutable data:
  `tuple`, `frozenset`, frozen pydantic models (`model_config = ConfigDict(frozen=True)`).
- Variable and class names should be descriptive and meaningful.
- Fields in pydantic models must have `description=` strings.
- Use **pytest** for all tests. Write tests early -- ideally before the code
  they test. Tests should be well-documented and not inhibit refactoring.
- Run the test suite frequently during development.

#### Guardrails

- Never use `eval()` or `exec()` on untrusted input.
- Validate all external input at system boundaries using pydantic models.
  Don't validate internal data that's already been validated at the boundary.
- Use `subprocess` with explicit argument lists, never `shell=True` with
  user-controlled input.
- Don't store secrets in code or config files committed to the repo.
  Use environment variables or a secrets manager.
- Prefer explicit over implicit -- avoid `**kwargs` swallowing unknown fields.
- Log errors at the right level; don't silently swallow exceptions.

### TypeScript / JavaScript

- Prefer strict TypeScript (`"strict": true` in tsconfig).
- Use `zod` for runtime validation of external data (API responses, user input),
  analogous to pydantic on the Python side.
- Avoid `any`. Use `unknown` and narrow it explicitly.
- No `eval()` or `new Function()` with dynamic strings.
- Sanitize any user-provided content before inserting into the DOM
  (use textContent, not innerHTML, unless you've explicitly sanitized).
- Use parameterized queries for any database access -- never string-interpolate
  SQL.
- Keep secrets server-side. Nothing sensitive in client bundles or localStorage.

---

## `.claude` Folder Layout

Claude Code uses two layers of configuration: user-level (`~/.claude/`) and
project-level (`.claude/` in the repo root).

### Project-level: `.claude/`

```
.claude/
  settings.json          ← shared team settings (commit this)
  settings.local.json    ← local overrides, personal permissions (gitignore this)
```

**`settings.json`** -- checked into the repo. Contains MCP server definitions
and shared tool permissions that all contributors should have:

```json
{
  "mcpServers": {
    "your-mcp-server": {
      "type": "sse",
      "url": "http://localhost:8001/sse"
    }
  },
  "permissions": {
    "allow": [
      "Bash(uv run:*)",
      "Bash(uv sync:*)",
      "Bash(uv pip:*)",
      "Bash(git add:*)",
      "Bash(git commit -m:*)"
    ]
  }
}
```

**`settings.local.json`** -- not committed. Personal tool permissions,
local MCP server URLs, dev-only overrides. Add to `.gitignore`.

### Project-level: `CLAUDE.md`

A markdown file at the repo root (or in subdirectories) that gives Claude
standing instructions for the project. Checked into the repo. Cover:

- Stack and tooling conventions (e.g. "use uv, not python3")
- Data model preferences (e.g. pydantic + immutability)
- Test requirements
- Architecture constraints
- Deployment targets

Subdirectory `CLAUDE.md` files are also read and apply to work within that
subtree -- useful for monorepos or packages with different conventions.

### User-level: `~/.claude/`

Managed by Claude Code. Key files:

```
~/.claude/
  settings.json          ← global MCP servers, global permissions
  CLAUDE.md              ← (optional) personal standing instructions
                            that apply across all projects
```

A personal `~/.claude/CLAUDE.md` is useful for preferences that apply
everywhere: preferred shell idioms, editor habits, communication style.
