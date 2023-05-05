```{mermaid}

    sequenceDiagram
      participant Alice
      participant Bob
      Alice->John: Hello John, how are you?
```

```{mermaid}
sequenceDiagram
    parse[parse-study-participants]->>dbquery[db-query]: QueryRequest = mergeStudyNode
    parse->>dbquery: QueryRequest = mergeParticipantNodes
    dbquery-)dbtriggers[db-triggers]: QueryResponse = mergeParticipantNodes
```
