---
layout: post
title:  "Advantages of Graph DB over RDBMS"
date:   2026-01-01 19:52:11 +02
categories: java spring neo4j graphdb
published: true
---
In Graph databases, data is stored as nodes and relationships instead of tables, forming a graph where nodes (vertices) connect via relationships (edges) in a labeled property graph.
Category of these nodes are called `Labels`.

#### OOP Analogy
Nodes act like objects, labels resemble interfaces, and properties function as object fields. Nodes can have multiple labels simultaneously—for instance, a *Bob* node might carry both `:Person` and `:Employee`; that's why labels are considered equivalent to `interfaces` rather than `classes`. Relationships denote meaningful, directed connections between nodes, such as `:DIRECTED_BY` linking `:Movie` and `:Director` nodes; in OOP terms, these often map to composition (e.g., a Movie field holding a Director) or inheritance.

#### Key Properties Feature

Properties attach to individual node or relationship instances, not labels, enabling schema flexibility where same-label nodes hold varying properties. This is a huge contrast compared to RDBMS where each entity would be part of a table schema and thus each entity will have all properties (represented as columns in RDBMS) and we are forced to assign `null`/default value for those which does not have a value. 
For example, in graph DB, it is possible to have a `Person` node with just name property and another `Person` node with name and age. Therefore, graph DB can be schemaless compared to RDBMS.

#### Performance Edge

Graph DBs embed direct pointers in nodes to relationships and vice versa, allowing traversal from a node to connected ones in O(1) steps
but in RDBMS this direct travel to its related node is not possible and it has to go via the relationship table (via JOIN QUERY).
Therefore, in Graph DB the total cost to get list of all related nodes of a node is log(#of nodes) + # of relations. For eg: If one social network has 1M users and `Bob` has 1000 friends, then to get details of friends of `Bob`, in  Graph DB, the operation will be the order of (log(1M) + 1000). If we have to do the same in a RDBMS, we have to use a query like below.
```
SELECT u2.* FROM Users u1 JOIN Friends f ON u1.id=f.user_id JOIN Users u2 ON f.friend_id=u2.id WHERE u1.name='BOB'
```
The same query will be order of 
 A: log(1M) -> To find id of Bob.
 B: 1B -> If 1 million users then we have to assume 1 billion rows in the friends table. Hence log(1B) to get index of `BOB` in friends table + 1000 to get his friend's ids.
 C:log(1M) -> To get each friends' details.
 Therefore, total operation cost will is on the order of (A + (B * C)) = O(log 1M + log 1B + 1000 * log 1M).
As we can see, when number of relationships increase (here friends), RDBMS' operational cost will dramatically increase. 

However, in graph DB, nodes store pointer to its related nodes. Therefore graph DB may require more memory because pointers will be stored in both source and destination and introduce redundancy which RDBMS doesn't have, as it stores as a single row for a relation.

Therefore, we can see that graph db can outperform RDBMS if the system can be represented as a nice graph and most of the operations need to process relations.

#### Free Beginner Courses
Neo4j offers these introductory courses for its graph DB:
  - [Neo4j Fundamentals](https://graphacademy.neo4j.com/courses/neo4j-fundamentals)
  - [Cypher Query Fundamentals](https://graphacademy.neo4j.com/courses/cypher-fundamentals/)
  - [Graph Data Modeling Fundamentals](https://graphacademy.neo4j.com/courses/modeling-fundamentals/)
  

**References**
1. [RDBMS vs GRAPHDB](https://aws.amazon.com/compare/the-difference-between-graph-and-relational-database/)
