---
template:
  id: https://www.modelware.io/sierra/system-analysis/ports
  name: "Ports"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Ports

Define the ports that constitute each component's interface and specify a direction for each. Every port should belong to exactly one component and declare a direction.

```table-editor
---
columns: { this: { label: "Port" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix oml: <http://opencaesar.io/oml#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:PortShape
    a sh:NodeShape ;
    sh:targetClass component:Port ;
    sh:property [
        sh:path component:portOf ;
        sh:name "Component" ;
        sh:class component:Component ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path component:direction ;
        sh:name "Direction" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 3 ;
    ] ;
    .
```

# Connections

Connect a source port to a target port and specify the items that flow accross it. A connection between peer components should run from an Out port to an In port, while a connection between a component and its container should have the same direction at both ends.

```table-editor
---
columns: { this: { label: "Connection" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix oml: <http://opencaesar.io/oml#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:ConnectionShape
    a sh:NodeShape ;
    sh:targetClass component:Connection ;
    sh:property [
        sh:path oml:hasSource ;
        sh:name "Source Port" ;
        sh:class component:Port ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path oml:hasTarget ;
        sh:name "Target Port" ;
        sh:class component:Port ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path component:transfers ;
        sh:name "Items" ;
        sh:class base:Item ;
        sh:order 3 ;
    ] ;
    sh:sparql [
        sh:message "A connection between peer components must run from an Out port to an In port." ;
        sh:select """
            SELECT $this WHERE {
                $this oml:hasSource ?src ; oml:hasTarget ?tgt .
                ?src component:direction ?srcDir ; component:portOf ?srcComp .
                ?tgt component:direction ?tgtDir ; component:portOf ?tgtComp .
                FILTER(?srcComp != ?tgtComp)
                FILTER NOT EXISTS { ?srcComp base:contains ?tgtComp }
                FILTER NOT EXISTS { ?tgtComp base:contains ?srcComp }
                FILTER(?srcDir != "Out" || ?tgtDir != "In")
            }
        """ ;
    ] ;
        sh:sparql [
        sh:message "A connection between a component and its container must have the same direction at both ends." ;
        sh:select """
            SELECT $this WHERE {
                $this oml:hasSource ?src ; oml:hasTarget ?tgt .
                ?src component:direction ?srcDir ; component:portOf ?srcComp .
                ?tgt component:direction ?tgtDir ; component:portOf ?tgtComp .
                { ?srcComp base:contains ?tgtComp } UNION { ?tgtComp base:contains ?srcComp }
                FILTER(?srcDir != ?tgtDir)
            }
        """ ;
    ] ;
    .
```

# Components 

The components these ports belong to. Edit them in the Components page.

```table-editor
---
columns: { this: { label: "Component" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:ComponentShape
    a sh:NodeShape ;
    sh:targetClass component:Component ;
    dash:readOnly true ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        sh:maxCount 1 ;
    ] ;
    .
```