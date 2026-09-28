---
template:
  id: https://www.modelware.io/sierra/system-analysis/port-wiring
  name: "Port Wiring"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: component
      type: iri
      required: true
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
### Port wiring of [[${component}]]

```table
PREFIX oml:       <http://opencaesar.io/oml#>
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?port ?direction ?connection ?otherPort ?wired
WHERE {
  <${component}> component:hasPort|^component:portOf ?port .
  ?port component:direction ?direction .
  OPTIONAL {
    ?connection a component:Connection .
    { ?connection oml:hasSource ?port ; oml:hasTarget ?otherPort }
    UNION
    { ?connection oml:hasTarget ?port ; oml:hasSource ?otherPort }
  }
  BIND(BOUND(?connection) AS ?wired)
}
ORDER BY ?wired ?port
```
