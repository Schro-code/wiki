---
domain: neo4j
tags: [neo4j]
sources:
  - "raw/Neo4j Fundamentals Pattern Matching  Querying Graphs.md"
updated: 2026-08-31
---

# Neo4j Fundamentals

## What it is

Neo4j is a native graph database: it stores data as nodes and relationships, not as tables and rows. Relationships are first-class citizens stored as physical pointers, not derived at query time through JOINs.

## What problem it solves

SQL databases become slow when data is highly connected — each additional hop between entities requires an extra JOIN, which grows exponentially. Neo4j solves this by making traversal O(1) per hop regardless of dataset size, because relationships are stored as direct pointers between nodes.

## Notes

### The three primitives

| Primitive | What it is | Example |
|---|---|---|
| **Node** | An entity or thing | `(:Person {name: "Ana", age: 30})` |
| **Relationship** | A connection between two nodes | `[:LIVES_IN {since: 2020}]` |
| **Property** | Key-value data on a node or relationship | `{name: "Ana"}` |

Relationships always have a **direction** and a **type**. Both nodes and relationships can carry properties. There is no fixed schema — nodes of the same label can have different properties.

### Node components

A node has three optional components: labels, properties, and an internal id.

**Labels** — classify the node (like a type or category). Prefixed with `:`. A node can have zero, one, or many labels.

```cypher
(:Person)              -- a person
(:City)                -- a city
(:Person:Employee)     -- a node that is both person and employee
```

**Properties** — key-value pairs that store data on the node. Any primitive type: string, integer, float, boolean, date, list.

```cypher
(:Person {name: "Ana", age: 30, active: true})
```

There is no fixed schema — nodes of the same label can have different properties.

---

### Relationship components

A relationship always has three required components and one optional.

| Component | Required | Example |
|---|---|---|
| **Type** | yes | `[:LIVES_IN]`, `[:ACTED_IN]` |
| **Direction** | yes | `->` (outgoing) or `<-` (incoming) |
| **Start and end node** | yes | connects exactly two nodes |
| **Properties** | no | `[:LIVES_IN {since: 2020}]` |

The type is written in `UPPER_SNAKE_CASE` by convention. Direction is always stored, but you can query without it when direction doesn't matter:

```cypher
-- with direction
MATCH (p:Person)-[:LIVES_IN]->(c:City)

-- without direction (matches either way)
MATCH (p:Person)-[:LIVES_IN]-(c:City)
```

### Cypher — the query language

Cypher uses ASCII-art patterns to describe what you want to find:

```
(node)-[:RELATIONSHIP]->(node)
```

**Basic operations:**

```cypher
-- Create a node
CREATE (:Person {name: "Ana", age: 30})

-- Create a relationship
MATCH (p:Person {name: "Ana"}), (c:City {name: "Madrid"})
CREATE (p)-[:LIVES_IN {since: 2020}]->(c)

-- Query: find all people who live in Madrid
MATCH (p:Person)-[:LIVES_IN]->(c:City {name: "Madrid"})
RETURN p.name

-- Query: friends of friends
MATCH (p:Person {name: "Ana"})-[:KNOWS*2]->(fof)
RETURN fof.name
```

The `*2` in the last example means "exactly 2 hops". Use `*1..3` for variable depth.

### Pattern matching syntax

Cypher uses ASCII-art to describe graph patterns. The building blocks:

| Element | Syntax | Meaning |
|---|---|---|
| Node | `(p:Person)` | Parentheses = node; `p` is a variable, `:Person` is the label |
| Relationship | `-[:ACTED_IN]->` | Square brackets = relationship; arrow = direction |
| Property filter | `{name: "Ana"}` | Inside node or relationship brackets |
| Variable | prefix before `:` | Used later in `WHERE` and `RETURN` |

A full pattern: `(p:Person)-[r:ACTED_IN]->(m:Movie)` — reads as "a Person node connected via an ACTED_IN relationship to a Movie node."

**`MATCH` with filter and return:**

```cypher
MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
WHERE p.name = 'Tom Hanks'
RETURN p.name, m.title, r.role
```

**Multiple hops in one pattern** — find co-actors of Tom Hanks:

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(p2:Person)
WHERE p.name = 'Tom Hanks'
RETURN p2.name AS actor, m.title AS movie
```

The `AS` keyword defines an alias for the returned value.

---

### SQL vs Neo4j — traversal cost

| Scenario | SQL | Neo4j |
|---|---|---|
| Direct connection | 1 JOIN | 1 hop |
| 2 degrees | 2 JOINs | 2 hops |
| 6 degrees of separation | 6 JOINs, very slow | 6 hops, fast |

## Relations

- [[graphrag/graph-construction]] — GraphRAG uses knowledge graphs built on top of graph database concepts; Neo4j is a common backend for production GraphRAG deployments

## Tags

#neo4j
