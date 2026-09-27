---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Assignment 5: Fire Force Analysis


## Orphan

**Question:** Which ports are not used by any connection?

**Why it matters:** A port with no connection is an interface that nothing talks to,
so either a connection is missing from the model or the port shouldn't exist.


```table
PREFIX oml:        <http://opencaesar.io/oml#>
PREFIX component:  <https://www.modelware.io/sierra/component#>

SELECT ?port ?outgoing ?incoming
WHERE {
  ?port a component:Port .
  OPTIONAL { ?port component:direction ?direction . }
  OPTIONAL { ?outgoing oml:hasSource ?port . }   # connections that START at this port
  OPTIONAL { ?incoming oml:hasTarget ?port . }   # connections that END at this port
  FILTER(!BOUND(?outgoing) && !BOUND(?incoming))
}

ORDER BY ?port

```

**Answer:** As it can be seen there are three components that don't have neither an outgoing nor incoming connections making them incomplete or the port to be unnecessary. 



## Conformance

**Question:** Does every connection respect the port-direction rule of the method?
Connections between peer components must go from an Out port to an In port;
connections between a component and its container must have the same direction at both ends.

**Why it matters:** A connection with the wrong directions describes a flow that can't
happen. Two outputs pushing into each other, or an input feeding an input, means
either the direction of a port is wrong or the connection is wired backwards. Either
way the interface definition can't be trusted for integration.

```table
PREFIX oml:       <http://opencaesar.io/oml#>
PREFIX base:      <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?connection ?fromComponent ?toComponent  ?isContained ?toDir ?fromDir ?conforms
WHERE {
  # 1. every connection and its two ports
  ?connection a component:Connection ;
              oml:hasSource ?fromPort ;
              oml:hasTarget ?toPort .

  # 2. the component that owns each port (either direction of the link)
  ?fromComponent component:hasPort|^component:portOf ?fromPort .
  ?toComponent   component:hasPort|^component:portOf ?toPort .

  # 3. the direction of each port
  ?fromPort component:direction ?fromDir . 
  ?toPort   component:direction ?toDir . 

  # 4. is one component inside the other? (true/false)
  BIND(EXISTS { ?fromComponent base:isContainedBy|^base:contains ?toComponent } ||
       EXISTS { ?toComponent   base:isContainedBy|^base:contains ?fromComponent }
       AS ?isContained)

  # 5. apply the method rule
  BIND(IF(?isContained,
          STR(?fromDir) = STR(?toDir),                    # child ↔ container: same direction
          STR(?fromDir) = "Out" && STR(?toDir) = "In")    # peers: Out → In
       AS ?conforms)
}
ORDER BY ?conforms ?connection
```


**Answer:** Indeed one is able to verify that there is conformance between the method and the actually implemented instances through the use of this query.

## Near miss

**Question:** Which components have some of their ports connected, but not all of them?

**Why it matters:** A component with no connections may simply not be wired yet.
A component that is *partly* wired is closer to done and more likely to have a
forgotten connection, since someone was already modeling its interfaces.

```table
PREFIX oml:       <http://opencaesar.io/oml#>
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?component
       (COUNT(?port) AS ?totalPorts)
       (SUM(?isConnected) AS ?connectedPorts)
       (SUM(?isConnected) / COUNT(?port) AS ?ratio)
WHERE {
  # 1. every (component, port) pair, once
  {
    SELECT DISTINCT ?component ?port
    WHERE { ?component component:hasPort|^component:portOf ?port . }
  }

  # 2. 1 if some connection uses this port, else 0
  BIND(IF(EXISTS { ?conn oml:hasSource|oml:hasTarget ?port }, 1, 0) AS ?isConnected)
}
GROUP BY ?component
# 3. keep only "partly connected": at least one, but not all
HAVING (SUM(?isConnected) > 0 && SUM(?isConnected) < COUNT(?port))
ORDER BY DESC(?ratio)
```

**Answer:** One component is a near miss: **PropulsionSegment** has 2 ports but only 1 is connected (ratio 0.5). `Power_In` receives power from the Platform, but `Command_In` is not used by any connection.

## Coverage

**Question:** Which entities provide each operational capability, and are there
capabilities that no entity provides?

**Why it matters:** A capability that no entity provides is a requirement with no
owner: the method says the system needs it, but nothing in the architecture
delivers it.

```matrix
---
rowColumnLabel: Capability / Entity
stylesheet:
  - selector: cell [Number(value) === 0]
    style:
      background-color: black
  - selector: cell [Number(value) > 0]
    style:
      background-color: darkblue
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity:  <https://www.modelware.io/sierra/entity#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  # 1. all rows: every capability
  { SELECT DISTINCT ?row WHERE { ?row a mission:Capability . } }

  # 2. all columns: every entity (Actors are entities too)
  { SELECT DISTINCT ?column WHERE {
      VALUES ?type { entity:Entity entity:Actor }
      ?column a ?type .
  } }

  # 3. 1 where the entity provides the capability; missing otherwise
  OPTIONAL {
    SELECT DISTINCT ?row ?column (1 AS ?n)
    WHERE { ?column entity:hasCapability|^entity:isAssignedTo ?row . }
  }
}
ORDER BY ?row ?column
```
**Answer:** As it is, the method is not being fully followed yet. One can observe that C9 has no entity providing the needed capability. Thus the query properly answers the question. The interesting finding is that neither lint, reason or validate find this gap. Thus illustrating the value of the queries.


## View Graph

**Question:** Can every mission objective be traced through a capability to an
entity that provides it?

**Why it matters:** Objectives are only achievable if the chain
objective → capability → entity is complete. A break anywhere in the chain means
an objective the architecture cannot deliver.


```graph
---
layout:
  mode: force
  running: true
  fit: true
  padding: 24
stylesheet:
  - selector: node [value.includes("/objectives#")]
    style:
      fill: gold
  - selector: node [value.includes("/capabilities#")]
    style:
      fill: cyan
  - selector: node [value.includes("/entities#") || value.includes("/stakeholders#")]
    style:
      fill: lightgreen
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity:  <https://www.modelware.io/sierra/entity#>

CONSTRUCT {
  ?objective mission:requires     ?capability .
  ?entity    entity:hasCapability ?capability .
}
WHERE {
  { ?objective mission:requires|^mission:isRequiredBy ?capability . }
  UNION
  { ?entity entity:hasCapability|^entity:isAssignedTo ?capability . }
}
```

**Broken chains only:** objectives that require a capability no entity provides,
or that require no capability at all.

```graph
---
layout:
  mode: force
  running: true
  fit: true
  padding: 24
stylesheet:
  - selector: node [value.includes("Objective") || outgoing.some(e => e.target.value.includes('Objective'))]
    style:
      fill: gold
  - selector: node [value.includes("Capability") || outgoing.some(e => e.target.value.includes('Capability'))]
    style:
      fill: tomato
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity:  <https://www.modelware.io/sierra/entity#>

CONSTRUCT {
  ?objective  a mission:Objective .
  ?capability a mission:Capability .
  ?objective  mission:requires ?capability .
}
WHERE {
  {
    # break 1: objective → capability, but no entity provides the capability
    ?objective mission:requires|^mission:isRequiredBy ?capability .
    FILTER NOT EXISTS { ?entity entity:hasCapability|^entity:isAssignedTo ?capability . }
  }
  UNION
  {
    # break 2: objective that requires no capability at all
    ?objective a mission:Objective .
    FILTER NOT EXISTS { ?objective mission:requires|^mission:isRequiredBy ?anyCapability . }
  }
}
```

**Answer:**The first graph illustrates the traceability from outcomes to capabilities and the entities providing those. The second view provides a graph that shows the broken chains. In this case capability 9 is missing an entity that provides it. By counter, every other capability has a direct entity providing the needed capabilities. 