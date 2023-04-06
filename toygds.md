# Simple GDS Examples

## Node Similarity Example

Use the following commands in the Neo4j Browser, an empty database. 
The GDS library must be installed.

### Step 1: creating some nodes and relationships

```
CREATE
(tom: Person {name: 'Tom'}),
(stefan: Person {name: 'Stefan'}),
(kristof: Person {name: 'Kristof'}),
(piet: Person {name: 'Piet'}),
(activision: Merchant {name: 'Activision'}),
(dustin: Merchant {name: 'Dustin'}),
(ebay: Merchant {name: 'eBay'}),
(amazon: Merchant {name: 'Amazon'}),
(hello_fresh: Merchant {name: 'HelloFresh'}),
(steam: Merchant {name: 'Steam'}),
(bikes: Merchant {name: 'Bikes.de'}),

(tom)-[:TRANSACTED_WITH {amount: 100}]->(amazon),
(tom)-[:TRANSACTED_WITH {amount: 50499}]->(dustin),
(tom)-[:TRANSACTED_WITH {amount: 220}]->(ebay),
(stefan)-[:TRANSACTED_WITH {amount: 220}]->(amazon),
(stefan)-[:TRANSACTED_WITH {amount: 399}]->(dustin),
(stefan)-[:TRANSACTED_WITH {amount: 1499}]->(ebay),
(stefan)-[:TRANSACTED_WITH {amount: 2200}]->(bikes),
(kristof)-[:TRANSACTED_WITH {amount: 423}]->(amazon),
(kristof)-[:TRANSACTED_WITH {amount: 530}]->(dustin),
(kristof)-[:TRANSACTED_WITH {amount: 1050}]->(hello_fresh),
(kristof)-[:TRANSACTED_WITH {amount: 230}]->(steam),
(kristof)-[:TRANSACTED_WITH {amount: 783}]->(activision),
(piet)-[:TRANSACTED_WITH {amount: 2100}]->(hello_fresh),
(piet)-[:TRANSACTED_WITH {amount: 230}]->(steam),
(piet)-[:TRANSACTED_WITH {amount: 783}]->(activision);
```

### Step 2: Make a GDS Graph Projection

We project now a BI-Partite graph with the name 'shopping'

```
CALL gds.graph.project('shopping',
["Person", "Merchant"],
{TRANSACTED_WITH: {properties: "amount"} })
```

### Step 3: Run a node similarity algorithm

```
CALL gds.nodeSimilarity.write('shopping', {
writeRelationshipType: 'IS_SIMILAR_TO',
writeProperty: 'SIM_SCORE',
relationshipWeightProperty: 'amount'
})
YIELD nodesCompared, relationshipsWritten
RETURN *
```

### Step 4: Cleanup duplicate relationships

```
MATCH (a:Person)-[r:IS_SIMILAR_TO]-&gt;(b:Person) WHERE (b)-[:IS_SIMILAR_TO]-&gt;(a) AND
id(a)&lt;id(b)
DELETE r
```
