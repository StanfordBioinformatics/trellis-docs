# Trellis v1.3 Update Design Doc

## Trellisdata Python Package

### Operation Grapher

* Added static methods \_is_query_valid(query) and \_is_trigger_valid(trigger) to the OperationGrapher class. These methods run on initialization of a new OperationGrapher instance and check for the presence and type of required and optional fields. Currently, there is no validation on the content.

## Database
### Relate Fastq read/mate pairs

The old query for getting Fastq mate pairs (paired-end sequencing) required matching Fastqs by sample and read group (composite index) and then collecting into different groups based on the "matePair" node property. This resulted in queries that were unwieldy and hard to decipher (decypher).

#### Database schema

Diagrams generated using [Sphinxcontrib-mermaid](https://sphinxcontrib-mermaid-demo.readthedocs.io/en/latest/) extension.

**Old model**
```{mermaid}
    graph TD
        seq[PersonalisSequencing] -- GENERATED --> r1[Fastq R1]
        seq --GENERATED --> r2[Fastq R2]
```

**New model**
```{mermaid}
    graph TD
        seq[PersonalisSequencing] -- GENERATED --> r1[Fastq R1]
        seq -- GENERATED --> r2[Fastq R2]
        r1 -- HAS_MATE_PAIR --> r2
```

#### Cypher query

**Old model** (from database-triggers.py)
```
MATCH (n:Fastq {sample: $sample, readGroup: $read_group})
WHERE NOT (n)-[:WAS_USED_BY]->(:JobRequest:FastqToUbam)
WITH n.sample AS sample,
     n.matePair AS matePair,
     COLLECT(n) AS matePairNodes
# In the case of duplicate Fastqs, only use one from each sequencing mate pair group.
WITH sample, COLLECT(head(matePairNodes)) AS uniqueMatePairs
# Check that there are a pair of Fastqs in the read group
WHERE size(uniqueMatePairs) = 2 
CREATE (j:JobRequest:FastqToUbam {
          sample:sample,
          nodeCreated: datetime(),
          nodeCreatedEpoch: datetime().epochSeconds,
          name: "fastq-to-ubam",
          eventId: $event_id })
WITH uniqueMatePairs, j, sample, j.eventId AS eventId
UNWIND uniqueMatePairs AS uniqueMatePair
MERGE (uniqueMatePair)-[:WAS_USED_BY]->(j)
RETURN DISTINCT(uniqueMatePairs) AS nodes
```

**New model**
```
MATCH (:PersonalisSequencing {sample: $sample})-[:GENERATED]->(r1:Fastq $read_group)-[rel:HAS_MATE_PAIR]->(r2:Fastq)
WHERE NOT (r1)-[:WAS_USED_BY]->(:JobRequest {name: "fastq-to-ubam"})
AND NOT (r2)-[:WAS_USED_BY]->(:JobRequest {name: "fastq-to-ubam"})
CREATE (job_request:JobRequest {
          name: 'fastq-to-ubam',
          sample: $sample,
          nodeCreated: datetime(),
          nodeCreatedEpoch: datetime().epochSeconds,
          eventId: $event_id)
WITH r1, rel, r2, job_request 
LIMIT 1
MERGE (r1)-[:WAS_USED_BY]->(job_request)
MERGE (r2)-[:WAS_USED_BY]->(job_request)
RETURN r1, rel, r2
```

