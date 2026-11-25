---
title: "What Is an Ontology? From Philosophical Roots to Knowledge Graphs, OWL, and the LLM Era"
date: 2026-11-25 14:00:00 +0000
categories:
- Computer Science
- Knowledge Representation
tags:
- Ontology
- Knowledge Graph
- Semantic Web
- RDF
- OWL
- SPARQL
- Description Logic
- Computational Semantics
- AI
---

In software engineering and artificial intelligence, few words are overloaded as heavily as **ontology**. Depending on who you ask, an ontology is either a lofty branch of metaphysics, a glorified database schema, a set of RDF/OWL files sitting in Apache Jena Fuseki, or the missing reasoning layer that stops Large Language Models (LLMs) from hallucinating.

At its core, however, an ontology has a very precise meaning in computer science. Just as [an array is defined by contiguous memory](/computer%20science/data%20structures/2026/09/14/definition-of-array/), an ontology is defined by **formal, machine-interpretable semantics**.

This post breaks down what an ontology actually is from first principles, how it differs from a taxonomy or a relational schema, how the Semantic Web stack (`RDF`, `RDFS`, `OWL`, `SPARQL`) implements it, and why ontologies have become critical again in the age of enterprise data platforms and LLMs.

---

## 1. The Definition: Philosophy vs. Computer Science

The word comes from the Greek *ontos* (being) + *logos* (study/discourse).

- **In Philosophy**, ontology is the study of *what exists* — the fundamental categories of being, properties, and relations in reality.
- **In Computer Science**, the classic and still unbeaten definition comes from Tom Gruber (1993):

> An **ontology** is an *"explicit, formal specification of a shared conceptualization."*

Every word in Gruber's five-word definition carries weight:

| Term | What It Actually Means |
|---|---|
| **Conceptualization** | An abstract model of a domain — identifying the relevant concepts (classes), entities, attributes, and relationships that exist in that world |
| **Explicit** | All concepts, constraints, and relationships are defined overtly; nothing is left hidden inside application code or naming conventions |
| **Formal** | Expressed in a mathematically rigorous, logic-based language ( typically First-Order Logic or **Description Logic**) so a machine can perform automated reasoning (inference) over it |
| **Shared** | Represents consensual knowledge across multiple systems, teams, or agents rather than an idiosyncratic view of a single table |

Remove **formal** and you just have documentation. Remove **explicit** and you have tribal knowledge. Remove **shared** and you have an isolated application schema.

---

## 2. The Semantic Spectrum: Taxonomy vs. Database Schema vs. Ontology

The easiest way to understand what an ontology does is to contrast it against structures engineers already use every day.

```
Low Semantic Expressiveness                           High Semantic Expressiveness
──────────────────────────────────────────────────────────────────────────────────►

  Controlled         Taxonomy           Relational /          Property           Formal
  Vocabulary      (Hierarchy Only)      JSON Schema            Graph            Ontology
  (Glossary)       (is-a tree)         (Structure/Types)     (LPG: Neo4j)      (OWL / DL)
```

### 1) Taxonomy vs. Ontology
A **taxonomy** organizes concepts into a single hierarchical tree using only parent-child (`is-a` / `subClassOf`) relationships:
- `ElectronicDevice -> Computer -> Laptop`

An **ontology** includes a taxonomy as its backbone, but adds **arbitrary typed relationships**, **properties**, and **logical axioms**:
- A `Laptop` *hasComponent* `Battery` (cardinality: $\ge 1$)
- `DefectiveProduct` $\equiv$ `Product` $\sqcap$ $\exists \text{hasReport}.\text{CriticalFailureReport}$
- `isCompatibleWith` is a *symmetric* property ($A \leftrightarrow B$); `isPartOf` is a *transitive* property ($A \to B \land B \to C \implies A \to C$).

### 2) Relational Database Schema vs. Ontology

Why isn't a PostgreSQL schema an ontology?

| Dimension | Relational Database Schema (SQL) | Formal Ontology (OWL / RDF) |
|---|---|---|
| **Primary Goal** | Data **validation** & storage layout integrity | Knowledge **representation** & automated **inference** |
| **World Assumption** | **Closed World Assumption (CWA)** — if a fact is not in the table, it is assumed *false* | **Open World Assumption (OWA)** — if a fact is not stated, it is simply *unknown* |
| **Constraints vs. Axioms** | Constraints reject invalid writes (`ERROR: violates check constraint`) | Axioms **infer new facts** from existing facts |
| **Schema Evolution** | Rigid (`ALTER TABLE` migrations) | Graph-native; classes and properties are first-class nodes that can be extended dynamically |
| **Identity** | Local primary keys (`id = 42` means nothing outside the DB) | Global URIs/IRIs (`http://example.org/onto#Product_42`) |

Consider a concrete example of **Constraint vs. Axiom**:
- Suppose we state the rule: *"Every Manager manages at least one Department,"* and we assert that *"Alice manages the AI Lab."*
- In a **relational database**, if you insert `Alice` into `managers` without a department, or into `employees` without setting `is_manager = true`, the database either throws a constraint error or requires manual application logic.
- In an **ontology**, if the axiom states `Domain(manages) = Manager` and `Range(manages) = Department`, and you assert the single triple `(Alice, manages, AILab)`, the **Reasoner automatically infers** that `Alice` is a `Manager` and `AILab` is a `Department` — even if you never explicitly labeled them as such.

---

## 3. Anatomy of an Ontology

Regardless of the domain — e-commerce catalogs, biomedical research (SNOMED CT, Gene Ontology), financial risk, or a personal memo knowledge base — every ontology is built from five building blocks:

```
                   ┌──────────────────────┐
                   │      Thing (⊤)       │
                   └──────────┬───────────┘
              subClassOf      │      subClassOf
         ┌────────────────────┴────────────────────┐
         ▼                                         ▼
┌─────────────────┐      writtenBy (ObjectProp)  ┌─────────────────┐
│   Class: Note   ├─────────────────────────────►│  Class: Person  │
└────────┬────────┘                              └────────┬────────┘
         │ rdf:type                                       │ rdf:type
         ▼                                                ▼
┌─────────────────┐        :writtenBy            ┌─────────────────┐
│  :Memo_2026_11  ├─────────────────────────────►│     :dhjin      │
└────────┬────────┘                              └─────────────────┘
         │ :createdAt (DatatypeProp)
         ▼
  "2026-11-25"^^xsd:date
```

1. **Classes (Concepts)**: Sets or categories of things (`Person`, `Note`, `Project`, `Task`). Classes form a hierarchy via `rdfs:subClassOf`.
2. **Individuals (Instances)**: Concrete real-world objects that belong to classes (`:Memo_2026_11`, `:dhjin`).
3. **Object Properties (Relations)**: Directed edges linking an Individual to another Individual (`:writtenBy`, `:dependsOn`, `:hasSubTask`).
4. **Datatype Properties (Attributes)**: Edges linking an Individual to a typed literal value (`:createdAt -> xsd:dateTime`, `:priority -> xsd:integer`).
5. **Axioms & Restrictions**: Logical rules governing classes and properties:
   - **Disjointness**: `CompletedTask` $\sqcap$ `PendingTask` $= \bot$ (an instance cannot be both at once).
   - **Property Characteristics**: `TransitiveProperty` (`ancestorOf`), `SymmetricProperty` (`colleagueOf`), `InverseOf` (`authorOf` $\equiv \text{writtenBy}^{-1}$).
   - **Equivalence & Restrictions**: Defining a class by necessary and sufficient conditions (`UrgentMemo` $\equiv$ `Memo` $\sqcap$ $\exists \text{hasTag}.\{\text{"URGENT"}\}$).

---

## 4. The Technical Stack: RDF, RDFS, OWL, and SPARQL

In practice, ontologies are most commonly implemented using the W3C Semantic Web stack, backed by triple stores like **Apache Jena Fuseki**, **GraphDB**, or **Blazegraph**.

```
┌────────────────────────────────────────────────────────┐
│  SPARQL (Query Language) & SHACL (Validation Shapes)   │
├────────────────────────────────────────────────────────┤
│  OWL 2 (Web Ontology Language — Description Logic)     │
│  Equivalence, Cardinality, Transitivity, Disjointness  │
├────────────────────────────────────────────────────────┤
│  RDFS (RDF Schema — Basic Vocabulary & Hierarchy)      │
│  Class, subClassOf, subPropertyOf, domain, range       │
├────────────────────────────────────────────────────────┤
│  RDF (Resource Description Framework — Triple Graph)   │
│  (Subject) ──[Predicate]──► (Object)                   │
├────────────────────────────────────────────────────────┤
│  URIs / IRIs & Serialization (Turtle, JSON-LD, RDF/XML)│
└────────────────────────────────────────────────────────┘
```

### Step 1: RDF (The Graph Data Model)
At the bottom is **RDF**, which represents all knowledge as atomic 3-tuples called **triples**:

$$\langle \text{Subject},\ \text{Predicate},\ \text{Object} \rangle$$

Written in **Turtle (`.ttl`)** syntax, an ontology plus instance data looks like this:

```turtle
@prefix :     <http://dhjin.github.io/ontology/memo#> .
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl:  <http://www.w3.org/2002/07/owl#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .

# --- 1. TBox (Terminological Box: Schema & Axioms) ---

:Item rdf:type owl:Class .

:Memo rdf:type owl:Class ;
      rdfs:subClassOf :Item .

:ActionableTask rdf:type owl:Class ;
                rdfs:subClassOf :Memo .

# Transitive dependency: if A blocks B and B blocks C, then A blocks C
:blocks rdf:type owl:ObjectProperty ,
                 owl:TransitiveProperty ;
        rdfs:domain :ActionableTask ;
        rdfs:range  :ActionableTask .

:hasDueDate rdf:type owl:DatatypeProperty ;
            rdfs:domain :ActionableTask ;
            rdfs:range  xsd:date .

# --- 2. ABox (Assertional Box: Instance Data) ---

:Task_DeployCluster rdf:type :ActionableTask ;
                    :hasDueDate "2026-11-30"^^xsd:date ;
                    :blocks :Task_RunBenchmark .

:Task_RunBenchmark  :blocks :Task_PublishPost .
```

Notice what we did **not** write in the ABox:
1. We never stated `:Task_RunBenchmark rdf:type :ActionableTask`.
2. We never stated `:Task_DeployCluster :blocks :Task_PublishPost`.
3. We never stated `:Task_DeployCluster rdf:type :Item`.

Yet when an OWL Reasoner (such as Pellet, HermiT, or Jena's OWLMicroReasoner) loads this file:
- From `rdfs:domain` / `rdfs:range` on `:blocks`, it infers that both `:Task_RunBenchmark` and `:Task_PublishPost` are instances of `:ActionableTask`.
- From `rdfs:subClassOf`, it infers all three are `:Memo` and `:Item`.
- From `owl:TransitiveProperty` on `:blocks`, it materializes the transitive edge `:Task_DeployCluster :blocks :Task_PublishPost`.

### Step 2: TBox vs. ABox (Description Logic)

Ontology engineers divide a knowledge base into two logical partitions:

$$\mathcal{KB} = \langle \mathcal{T},\ \mathcal{A} \rangle$$

- **TBox (Terminological Box, $\mathcal{T}$)**: The ontology itself — class definitions, hierarchies, and property axioms (`Memo` $\sqsubseteq$ `Item`, `Transitive(blocks)`).
- **ABox (Assertional Box, $\mathcal{A}$)**: The factual instance data (`ActionableTask(Task_DeployCluster)`).

A **Knowledge Graph** is simply an ontology ($\mathcal{T}$) populated with large-scale instance facts ($\mathcal{A}$).

### Step 3: Querying with SPARQL

Once the TBox and ABox are loaded into an RDF triple store (like **Apache Jena Fuseki**), you query both explicit and inferred facts via **SPARQL**:

```sparql
PREFIX : <http://dhjin.github.io/ontology/memo#>

SELECT ?upstreamTask ?downstreamTask ?dueDate
WHERE {
    ?upstreamTask a :Memo ;
                  :blocks ?downstreamTask ;
                  :hasDueDate ?dueDate .
}
ORDER BY ?dueDate
```

Because reasoning is handled by the ontology engine, the application query stays simple: you do not need recursive SQL `WITH RECURSIVE` CTEs to traverse `:blocks`, nor complex `JOIN` trees to check subclasses of `:Memo`.

---

## 5. Why Ontologies Matter Again in 2026: Data Platforms & LLMs

For a while in the 2010s, critics dismissed formal ontologies as overly academic — arguing that scale-out NoSQL, data lakes, and deep learning would replace explicit symbolic modeling.

In 2026, the exact opposite has happened. Ontologies have returned to the center of system design for two practical reasons:

### 1) The Semantic Layer of Enterprise Data Platforms
Modern organizations have thousands of tables scattered across Iceberg data lakes, StarRocks OLAP clusters, Kafka streams, and microservices. Without a shared ontology:
- Marketing defines `ActiveCustomer` as *"logged in within 30 days."*
- Finance defines `ActiveCustomer` as *"paid an invoice within 90 days."*
- E-commerce catalog pipelines struggle to unify vendor categories (`"Notebook PC"`, `"Laptops"`, `"MacBook"`) and product attributes.

An ontology acts as the **domain semantic layer**, decoupling *business meaning* from *physical table storage*. Systems like Palantir Foundry built their entire operational paradigm around mapping raw data lake tables into a central enterprise **Ontology** of Objects, Properties, Links, and Actions.

### 2) Grounding LLMs: GraphRAG & Neuro-Symbolic AI
Large Language Models excel at linguistic fluency and unstructured text extraction, but they struggle with:
- **Multi-hop deterministic reasoning** (e.g., traversing 5 levels of supply-chain or task dependencies without hallucinating)
- **Strict domain rules and schema compliance**

```
Unstructured Text / User Query
              │
              ▼
┌───────────────────────────┐      Extracts Entities / Generates SPARQL
│  Large Language Model     ├──────────────────────────────────────────┐
│  (Statistical / Neural)   │◄─────────────────────────────────────────┤
└───────────────────────────┘      Returns Verified Subgraph & Axioms  │
                                                                       ▼
                                                         ┌───────────────────────────┐
                                                         │  Ontology + Knowledge     │
                                                         │  Graph (Symbolic / Logic) │
                                                         └───────────────────────────┘
```

Pairing an LLM with an ontology creates a **Neuro-Symbolic architecture**:
1. **Ontology-Guided Extraction**: When building a Knowledge Graph from unstructured documents or memos, the ontology's TBox constrains the LLM so it only extracts valid classes and predicates rather than inventing arbitrary, inconsistent edge names.
2. **Deterministic GraphRAG**: Instead of relying purely on fuzzy vector-embedding similarity (`top-k` chunks), the agent translates the user's question into a traversal over the ontology graph, using formal axioms (subclasses, transitivity, inverse relations) to retrieve exact, provenance-backed facts.

---

## Summary

- An **ontology** is an *explicit, formal specification of a shared conceptualization* — it encodes not just data structure, but **domain meaning and logical rules**.
- Unlike a **taxonomy** (which only models `is-a` trees) or a **relational schema** (which enforces closed-world storage constraints), an ontology uses **Description Logic** (`RDF` / `RDFS` / `OWL`) to **infer new knowledge** from existing facts.
- Dividing knowledge into a **TBox** (conceptual axioms) and an **ABox** (instance assertions) allows triple stores like Apache Jena Fuseki to answer complex semantic queries via **SPARQL** without hardcoding domain rules in application code.
- In the era of Lakehouses and LLMs, ontologies provide the **symbolic backbone** needed to unify enterprise data semantics and ground generative AI in verifiable logic.
