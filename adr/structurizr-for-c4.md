---
status: "live"
initial_date: "2026-10-29"
---

# ADR: Adopt Structurizr DSL for C4 Architecture Modeling

## Context and Problem Statement
Architectural documentation across our repositories suffers from visual drift and broken synchronization when maintained inside manual drag-and-drop drawing tools. We need a standard way to author C4 architecture models (System Context, Container, and Component views) that lives directly inside our Git repositories, supports clean line-by-line pull request comparisons, and allows AI code assistants to generate and validate topology definitions without inventing fake visual coordinates. At the same time, the chosen format must minimize authoring boilerplate so engineers can write and commit architecture updates immediately without defining heavy JSON schemas first.

## Decision Drivers
* **AI Generation Compatibility and Low Token Overhead:** The syntax must have a compact, deterministic grammar that large language models can generate, refactor, and validate via command-line tools without wasting context tokens on visual styling metadata.
* **Single-Model Multi-View Reuse:** A single definition of system entities and relationships must project across multiple C4 abstraction levels (Context, Container, and Component) without duplicating node declarations across separate files.
* **Readable Version Control Comparisons and Direct Repository Storage:** Architecture source files must be human-readable plain text that can be committed directly to the codebase and reviewed in standard Git pull requests without XML coordinate noise.
* **Low Authoring Friction for Immediate Adoption:** Engineers must be able to author and render C4 diagrams directly from the repository with minimal upfront schema setup.

## Considered Options
* **Structurizr DSL**
* **diagrams.net (Draw.io)**
* **FINOS CALM (Common Architecture Language Model)**

## Decision Outcome
Chosen option: "**Structurizr DSL**," because it separates the semantic C4 architecture model from visual layout views in a concise plain-text grammar (`workspace.dsl`), is natively supported by AI code generators and official language tools, and commits directly to Git with zero JSON-schema or coordinate boilerplate.

### Benefits
* Good, because entities (`person`, `softwareSystem`, `container`, `component`) and relationships (`->`) are declared once in the `model` block and reused across multiple `views` blocks (`systemContext`, `container`, `component`, `deployment`), preventing inconsistency between C4 zoom levels.
* Good, because the concise declarative syntax requires minimal tokens and zero geometry properties, allowing AI assistants to generate valid C4 models from codebase context and verify them via `structurizr.sh validate`.
* Good, because `.dsl` files are plain text and modular via `!include` and `!identifiers hierarchical`, producing clean, readable line changes during pull request reviews.
* Good, because the Structurizr command-line tool (`export`) can compile the same `.dsl` source into Mermaid, PlantUML, or static HTML sites directly inside continuous integration pipelines.

### Trade-offs
* Bad, because automated graph layout engines (`autoLayout`) offer limited manual control over exact box positioning and edge routing compared to freeform canvas tools.
* Bad, because Structurizr DSL focuses on visualization and structural documentation rather than enforcing strict JSON Schema policy controls unless paired with custom parser scripts.

### Confirmation
Compliance is enforced through repository structure and continuous integration validation gates:
1. **Syntax Validation Gate:** Pull requests modifying `*.dsl` files execute the Structurizr validation check (`structurizr validate -workspace workspace.dsl`) in GitHub Actions to reject broken relationships or invalid C4 hierarchy references.
2. **File Format Gate:** Repository pre-commit and linter rules (`ls-lint`) require architecture definitions under `docs/architecture/` to use `.dsl` extensions and forbid uncommitted binary image exports or raw `.drawio` XML files as the primary source of truth.

## Pros and Cons of the Options

### Structurizr DSL
A domain-specific language created by Simon Brown specifically for authoring C4 software architecture models as code, separating model semantics from diagram views.

* Good, because a single `.dsl` file defines the entire C4 hierarchy (`softwareSystem` -> `container` -> `component`) and renders multiple filtered diagrams without duplicating node definitions.
* Good, because AI agents can output valid Structurizr DSL in a single pass and validate the syntax tree through the standalone command-line parser.
* Good, because it supports native decision record linking (`!adrs`) and Markdown documentation (`!docs`) packaged alongside the architecture workspace.
* Neutral, because rendering interactive diagrams locally requires running the local Structurizr Docker container (`structurizr/structurizr`) or exporting views to Mermaid and PlantUML via the command line.
* Bad, because manual layout adjustments store coordinates in a separate `workspace.json` file, which can drift if not regenerated cleanly.

### diagrams.net (Draw.io)
A general-purpose visual diagramming canvas that stores shapes, connectors, and C4 stencil metadata inside `.drawio` (`mxGraphModel` XML) or `.drawio.svg` files.

* Good, because it gives authors pixel-level manual control over box placement, connector waypoints, and custom visual annotations.
* Bad, because each C4 level (Context, Container, Component) is drawn as an independent visual canvas; renaming a container on the Container diagram does not automatically update its references on the Component diagram, causing documentation drift.
* Bad, because `.drawio` files serialize into thousands of lines of nested XML containing geometry and coordinate attributes, making Git pull request comparisons unreadable and causing merge conflicts.
* Bad, because AI generators struggle to calculate spatial coordinates and non-overlapping connector paths in raw XML, regularly producing overlapping boxes or broken XML trees.

### FINOS CALM (Common Architecture Language Model)
An open-source JSON Schema specification maintained by FINOS that defines software architectures as structured JSON data (`nodes`, `relationships`, `metadata`, and `controls`) for automated governance.

* Good, because it validates architectural topologies and compliance controls against strict JSON Schema definitions inside continuous integration pipelines using `@finos/calm-cli`.
* Good, because its structured JSON format is machine-readable and well-suited for automated policy enforcement and enterprise risk governance.
* Neutral, because it models architecture as a general graph of nodes and interfaces rather than enforcing opinionated C4 view hierarchies out of the box.
* Bad, because authoring and maintaining raw `.architecture.json` files directly in a repository is significantly more verbose than Structurizr DSL, requiring explicit `unique-id`, `node-type`, `interfaces`, and `connects` blocks for every relationship.
