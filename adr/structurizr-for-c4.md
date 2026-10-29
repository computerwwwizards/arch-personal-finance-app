---
status: "live"
initial_date: "2026-10-29"
---

# ADR: Adopt Structurizr DSL for C4 Architecture Modeling

## Context and Problem Statement
Architectural documentation across our repositories suffers from visual drift and broken synchronization when maintained inside manual drag-and-drop drawing tools. We need a standard way to author C4 architecture models (System Context, Container, and Component views) that lives directly inside our Git repositories, supports clean line-by-line pull request diffs, and allows AI code assistants and Model Context Protocol (MCP) tools to generate and validate topology definitions without hallucinating visual coordinates. At the same time, the chosen format must minimize authoring boilerplate so engineers can write and commit architecture updates immediately without defining heavy JSON schemas first.

## Decision Drivers
* **AI Generation Compatibility and Low Token Overhead:** The syntax must have a compact, deterministic grammar that large language models (LLMs) can generate, refactor, and validate via CLI or MCP servers without wasting context tokens on visual styling metadata.
* **Single-Model Multi-View Reuse:** A single definition of system entities and relationships must project across multiple C4 abstraction levels (L1 Context, L2 Container, L3 Component) without duplicating node declarations across separate files.
* **Git-Native Diffability and Direct Repository Storage:** Architecture source files must be human-readable plain text that can be committed directly to the codebase and reviewed in standard Git pull requests without XML coordinate noise.
* **Low Authoring Friction for Immediate Adoption:** Engineers must be able to author and render C4 diagrams directly from the repository with minimal upfront schema setup.

## Considered Options
* **Structurizr DSL**
* **diagrams.net (Draw.io)**
* **FINOS CALM (Common Architecture Language Model)**

## Decision Outcome
Chosen option: "**Structurizr DSL**," because it separates the semantic C4 architecture model from visual layout views in a concise plain-text grammar (`workspace.dsl`), is natively supported by LLM code generators and the official Structurizr MCP server, and commits directly to Git with zero JSON-schema or coordinate boilerplate.

### Benefits
* Good, because entities (`person`, `softwareSystem`, `container`, `component`) and relationships (`->`) are declared once in the `model` block and reused across multiple `views` blocks (`systemContext`, `container`, `component`, `deployment`), preventing inconsistency between C4 zoom levels.
* Good, because the concise declarative syntax requires minimal tokens and zero `x/y` geometry properties, allowing AI assistants to generate valid C4 models from codebase context and verify them via `structurizr.sh validate` or the Structurizr MCP server.
* Good, because `.dsl` files are plain text and modular via `!include` and `!identifiers hierarchical`, producing clean, readable diffs during pull request reviews.
* Good, because the Structurizr CLI (`export`) can compile the same `.dsl` source into Mermaid, PlantUML, or static HTML sites directly inside CI/CD pipelines.

### Trade-offs
* Bad, because automated graph layout engines (Graphviz/Dagre via `autoLayout`) offer limited manual control over exact box positioning and edge routing compared to freeform canvas tools.
* Bad, because Structurizr DSL focuses on visualization and structural documentation rather than enforcing strict JSON Schema policy controls unless paired with custom parser scripts.

### Confirmation
Compliance is enforced through repository structure and CI validation gates:
1. **DSL Syntax Gate:** Pull requests modifying `*.dsl` files execute the Structurizr CLI validation check (`structurizr validate -workspace workspace.dsl`) in GitHub Actions to reject broken relationships or invalid C4 hierarchy references.
2. **File Format Gate:** Repository pre-commit and linter rules (`ls-lint`) require architecture definitions under `docs/architecture/` to use `.dsl` extensions and forbid uncommitted binary image exports or raw `.drawio` XML files as the primary source of truth.

## Pros and Cons of the Options

### Structurizr DSL
A domain-specific language created by Simon Brown specifically for authoring C4 software architecture models as code, separating model semantics from diagram views.

* Good, because a single `.dsl` file defines the entire C4 hierarchy (`softwareSystem` -> `container` -> `component`) and renders multiple filtered diagrams without duplicating node definitions.
* Good, because LLM agents can output valid Structurizr DSL in a single pass and validate the AST through the standalone open-source Structurizr MCP server.
* Good, because it supports native ADR linking (`!adrs`) and Markdown documentation (`!docs`) packaged alongside the architecture workspace.
* Neutral, because rendering interactive diagrams locally requires running the Structurizr local container (`structurizr/structurizr`) or exporting views to Mermaid/PlantUML via CLI.
* Bad, because manual layout adjustments (`autolayout` overrides) store coordinates in a separate `workspace.json` file, which can drift if not regenerated cleanly.

### diagrams.net (Draw.io)
A general-purpose visual diagramming canvas that stores shapes, connectors, and C4 stencil metadata inside `.drawio` (`mxGraphModel` XML) or `.drawio.svg` files.

* Good, because it gives authors pixel-level manual control over box placement, connector waypoints, and custom visual annotations.
* Bad, because each C4 level (Context, Container, Component) is drawn as an independent visual canvas; renaming a container on the L2 diagram does not automatically update its references on the L3 diagram, causing documentation drift.
* Bad, because `.drawio` files serialize into thousands of lines of nested XML containing `mxCell`, `mxGeometry`, and `x/y` coordinate attributes, making Git pull request diffs unreadable and causing merge conflicts.
* Bad, because AI generators struggle to calculate spatial coordinates and non-overlapping connector paths in raw `mxGraphModel` XML, regularly producing overlapping boxes or broken XML trees.

### FINOS CALM (Common Architecture Language Model)
An open-source JSON Schema specification maintained by FINOS that defines software architectures as structured JSON data (`nodes`, `relationships`, `metadata`, and `controls`) for automated governance.

* Good, because it validates architectural topologies and compliance controls against strict JSON Schema definitions inside CI/CD pipelines using `@finos/calm-cli`.
* Good, because its structured JSON format is machine-readable and well-suited for automated policy enforcement and enterprise risk governance.
* Neutral, because it models architecture as a general graph of nodes and interfaces rather than enforcing opinionated C4 view hierarchies out of the box.
* Bad, because authoring and maintaining raw `.architecture.json` files directly in a repository is significantly more verbose than Structurizr DSL, requiring explicit `unique-id`, `node-type`, `interfaces`, and `connects` blocks for every relationship.
* Bad, because rendering human-readable C4 views from CALM JSON requires an additional translation step (`calm-cli` visualization generators or CALM Studio) rather than providing a direct, self-contained C4 view syntax.
