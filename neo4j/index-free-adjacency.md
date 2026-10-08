---
domain: neo4j
tags: [neo4j]
sources:
  - raw/Neo4j Fundamentals Thinking in Graphs  Graph Thinking.md
updated: 2026-08-31
---

# Index-Free Adjacency

## What it is

Index-free adjacency is Neo4j's storage mechanism where each node holds direct memory pointers to its neighboring nodes. Traversing a relationship means following a pointer — no index is consulted.

## What problem it solves

Relational databases compute joins at read-time using indexes. As data grows, those indexes grow too, making join queries progressively slower. This degradation is called the **O(n) problem**.

## Notes

### The O(n) problem in relational databases

In Big O notation, `n` is the size of your data. O(n) means performance scales linearly with data size: double the data, double the time.

When you join two tables in SQL, the database looks up matching rows using an **index** — a sorted data structure (typically a B-tree) that maps column values to row locations.

**Concrete example: Users and Orders**

```
Table: users                   Table: orders
-----------------------        ---------------------------------
user_id | name                 order_id | user_id | product
--------|-------               ---------|---------|--------
1       | Alice                101      | 1       | Laptop
2       | Bob                  102      | 2       | Phone
3       | Carol                103      | 1       | Mouse
                               104      | 3       | Keyboard
```

Query: *"Give me all of Alice's orders."*

```sql
SELECT u.name, o.product
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.name = 'Alice'
```

**What the database actually does:**

1. Find Alice in `users` → `user_id = 1`.
2. Look up `1` in the index on `orders.user_id`:

```
Index on orders.user_id (B-tree):
  1 → [row 101, row 103]
  2 → [row 102]
  3 → [row 104]
```

3. Retrieve rows 101 and 103. Done.

This is fast when the table is small. The index is the problem.

**How the index grows:**

```
Year 1:  500,000 orders   → index has 500,000 entries  → fits in RAM, fast
Year 2:  10,000,000 orders → index has 10M entries      → starts hitting disk
Year 3:  200,000,000 orders → index has 200M entries    → disk reads, slow
```

The index is not the data — it is a separate structure maintained by the database. Searching it costs O(log n) per lookup (binary search on a B-tree). With multiple joins, the cost compounds:

```sql
SELECT u.name, o.product, p.category, r.score
FROM users u
JOIN orders o ON u.user_id = o.user_id        -- index lookup 1
JOIN products p ON o.product_id = p.product_id -- index lookup 2
JOIN reviews r ON o.order_id = r.order_id      -- index lookup 3
WHERE u.name = 'Alice'
```

Four tables = three index lookups, each growing independently. As data scales, every join adds latency.

---

### How Neo4j solves it

When Neo4j writes a relationship between two nodes, it stores a **direct pointer** from each node to the other. No index is created or consulted at read-time.

The same data in Neo4j:

```
(Alice:User) --[:PLACED]--> (Order:101 {product: "Laptop"})
(Alice:User) --[:PLACED]--> (Order:103 {product: "Mouse"})
(Bob:User)   --[:PLACED]--> (Order:102 {product: "Phone"})
```

Query in Cypher:

```cypher
MATCH (u:User {name: "Alice"})-[:PLACED]->(o:Order)
RETURN u.name, o.product
```

**What the database actually does:**

1. Find the Alice node.
2. Follow the `PLACED` pointers stored directly on the node → lands on Order 101 and Order 103.
3. Return properties.

No index traversal. No growing structure. The cost of step 2 is **O(1) per hop**, regardless of how many other users or orders exist in the database.

**Scaling comparison:**

| | 500K orders | 10M orders | 200M orders |
|---|---|---|---|
| SQL JOIN | Fast | Slower | Slow |
| Neo4j traversal | Fast | Fast | Fast |

Adding a billion unrelated nodes does not affect the speed of traversing Alice's two orders.

---

### When this matters most

- **Deep hierarchies**: "Who does Aisha ultimately report to?" requires traversing an unknown number of levels. SQL needs recursive CTEs and grows expensive. Cypher handles it with `*` (variable-depth):

```cypher
MATCH (e:Employee {name: "Aisha"})-[:REPORTS_TO*]->(boss)
RETURN boss.name
```

- **Path finding**: shortest path through a transport network, friend-of-friend recommendations.
- **Variable-depth parts explosions**: a recalled component traced through all assemblies that contain it, at any depth.

## Relations

- [[neo4j-fundamentals]] — the core data model (nodes, relationships, properties) that makes index-free adjacency possible

## Tags

#neo4j
