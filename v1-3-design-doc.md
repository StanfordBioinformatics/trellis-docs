# Trellis v1.3 Update Design Doc

## Functions

### Using a single function to launch jobs

Current method for adding new bioinformatics (or other) tasks to Trellis is to create a new Cloud Function specifically tailored to launch jobs of that type of task (e.g. "samtools flagstat"). Limitations of generating separate functions for each task include:
  
  * Copying of a lot of boilerplate code across functions
  * Potential to differences in boilerplate code across functions
  * Changing mechanisms for launching jobs requires changing every launcher function
  * Creating a new launcher function is kind of an obtuse process, requiring knowledge of Python and the 'trellisdata' package. If you didn't have an example to look at it, it would be a huge pain.

**Pre Trellis v1.3 architecture**
```{mermaid}
    graph TD
        gcs[CloudStorage] -- TRIGGERS --> create-blob-node
        create-blob-node -- QUERY-REQUEST --> db-query
        db-query -- QUERY-RESULT --> check-triggers
        check-triggers -- QUERY-REQUEST --> db-query
        db-query -- QUERY-RESULT --> job-launcher
        job-launcher -- JOB-RESULT --> create-job-node
        create-job-node -- QUERY-REQUEST --> db-query
```


#### What are the elements of a standard launcher function?

**Inputs.** Inputs to all jobs are nodes or relationship structures: (node)-[relationship]->(node). I already define these patterns in a standard way in the database trigger YAML, so I can replicate that in a launcher configuration file. And, I think YAML configuration files are going to be the default method for configuring Trellis.

* Database trigger configuration: database-triggers.yaml
* Database query configuration: database-queries.yaml
* Job launcher configuration: job-launchers.yaml

Also, in regards to terminology I'm thinking of a **job** as an instance of a **task**. 

**Trellis configuration.** Information that is stored in the Trellis configuration object and is uniform across Trellis (e.g. project, regions, buckets).

**Dsub configuration.** This might be the most challenging part; defining the input, output, and environment variables.

**Virtual machine configuration.** Information regarding CPUs, memory, and disk size can vary between tasks and should be specificied in the configuration of each task.

#### Jobs are going to have different numbers of nodes as inputs.

* fastq-to-ubam: 2 nodes connected by relationship
* gatk-5-dollar: 2-16 nodes, not connected
* extract-mapped-reads: 1 node

#### How do I know which job to launch?
When every job had its own launcher function, the job was determined by the pubsub topic that the database query result was published to. The topic was defined as part of the database query. How will I choose the job if all query results are routed throug the same function?

I could update the QueryResponse classes to also include a field with the task to be launched.

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

### Incorporating OMOP graph model
One of the challenges we face when expanding the scope of our use cases is how to incorporate phenotype data into the database. The Observational Medical Outcomes Parternships (OMOP) Common Data Model (CDM) is used by the [Stanford Research Data Repository (STARR)](https://med.stanford.edu/starr-omop.html) and seemed like a good place to start. Unfortunately, OMOP is designed for relational databases. However, a group at Northwestern University has drafted a [graph model](https://github.com/NUSCRIPT/OMOP_to_Graph) implementation of the OMOP CDM.

Ours is a research use case, as opposed to clinical, and we don't have much of the data that would go into a clinical database, so it's not clear how useful this model will be for us. But, I also don't know of a better alternative, so we are going to start from this model and see how it goes.

Right now we are only using a small set of the nodes and relationships described by the graph OMOP CDM.

* Node labels (5): Concept, ConceptClass, Domain, Vocabulary, ConditionOccurrence
* Relationship types (5): HAS_CONCEPT, BELONGS_TO_CLASS, USES_VOCABULARY, IN_DOMAIN, HAS_CONDITION_OCCURRENCE

**Subset of OMOP graph model added to Trellis**
```{mermaid}
    graph TD
        person[Person] -- HAS_CONDITION_OCCURRENCE --> occurrence[Condition Occurrence]
        occurrence -- HAS_CONCEPT --> concept[Concept]
        concept -- IN_DOMAIN --> domain[Domain]
        domain -- HAS_CONCEPT --> concept
        concept -- BELONGS_TO_CLASS --> conceptclass[Concept Class]
        conceptclass -- HAS_CONCEPT --> concept
        concept -- USES_VOCABULARY --> vocab[Vocabulary]
        vocab -- HAS_CONCEPT --> concept
```

I sometimes find it difficult to interpret OMOP because the model is so abstract. But that's the point, it's general enough to fit all kinds of medical information. Right now, we are only using this model subset to describe phenotypes associated with Abdominal Aortic Aneurysm (AAA), so let's look what the AAA instances of these nodes look like.

**Example of AAA diagnosis represented with OMOP**
```{mermaid}
    graph TD
        person[Person] -- HAS_CONDITION_OCCURRENCE --> occurrence[Condition Occurrence]
        occurrence -- HAS_CONCEPT --> concept[Abdominal Aortic Aneurysm]
        concept -- IN_DOMAIN --> domain[Condition]
        domain -- HAS_CONCEPT --> concept
        concept -- BELONGS_TO_CLASS --> conceptclass[Clinical Finding]
        conceptclass -- HAS_CONCEPT --> concept
        concept -- USES_VOCABULARY --> vocab[SNOMED]
        vocab -- HAS_CONCEPT --> concept
```

The script I used to update the database with AAA data can be found in the [trellis-mvp-data-modelling](https://github.com/va-big-data-genomics/trellis-mvp-data-modelling/blob/main/database-update-scripts/add-aaa-phenotypes.py) repository.

=======
# Node label ontology
```{mermaid}
    graph TD
        root --> Blob
        Blob --> Fastq
        Blob --> Index
        root --> Person
        root --> GcpInstance
        GcpInstance --> CromwellAttemtp
        GcpInstance --> DsubJob
```

## Deployment configuration
Previously, the Terraform files for configuring Trellis were stored in a separate repository. I don't think that's a good setup. The deployment configuration and application version are intricately linked and should be tracked together. Todo: Add the Terraform resources to the Trellis functions repository.

