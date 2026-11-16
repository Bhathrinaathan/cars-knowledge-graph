# CarsKG — Automotive Knowledge Graph

A semantic-web knowledge graph of the car industry: parent companies, the manufacturers they own, the models those manufacturers produce, each model's engine, body and performance specifications, three years of sales figures, and which models compete with which.

Built end to end with the standard Linked Data stack and loaded into GraphDB:

| Layer | What is here |
|---|---|
| **Ontology** | OWL 2 in Turtle (`OWL_ontology.ttl`, `cars_ontology.ttl`), 8 core classes aligned to `schema:Organization`, `schema:Vehicle` and `schema:Country` |
| **Data uplift** | RML mapping (`rml_mapping.ttl`) from `data/cars.csv` to RDF, run with the RMLMapper; 28 car models across 6 parent groups (BMW, Mercedes-Benz, Porsche, Suzuki, Toyota, Volkswagen) |
| **Linking** | `data/competitors.ttl` adds symmetric `competitorOf` relationships; manufacturers and countries are aligned to DBpedia and Wikidata resources |
| **Querying** | 7 SPARQL queries (`sparql.ttl`): aggregation, property paths, sub-queries, UNION, REGEX filtering |
| **Validation** | 7 SHACL node shapes (`shacl.ttl`) covering cardinality, datatypes and required properties for every class |
| **Diagrams** | `docs/` holds the ontology diagram (PDF) and the editable draw.io source |

Built for the Knowledge and Data Engineering module (CS7IS1), MSc Computer Science (Intelligent Systems), Trinity College Dublin, 2025.

## Ontology at a glance

```
ParentCompany ──owns──▶ Manufacturer ──produces──▶ Car ──hasSpecification──▶ Specifications
                          │ partOf (inverse)         │                          ├─ Engine
                          │                          ├─ hasSalesRecord ──▶ SalesRecord (2023–2025)
                          └─ countryOfOrigin ──▶ Country   └─ competitorOf ◀──▶ Car (symmetric)
```

`partOf` / `owns`, `produces` / `manufacturedBy` and the specification links are declared as OWL inverse pairs, `controls` is an `owl:TransitiveProperty` so a query can climb from a model to its ultimate parent group in one property path, and `competitorOf` is an `owl:SymmetricProperty` so a single asserted edge answers "who competes with X" in either direction.

## Example queries

Market share by parent group over three years, from `sparql.ttl`:

```sparql
SELECT ?companyName (SUM(?sales2023 + ?sales2024 + ?sales2025) AS ?totalSales)
       (AVG(?sales2024) AS ?avgSales2024) (COUNT(?car) AS ?numberOfModels)
WHERE {
  ?car a ex:Car ; ex:manufacturedBy ?manufacturer ; ex:hasSalesRecord ?salesRec .
  ?manufacturer ex:partOf ?company .
  ?salesRec ex:sales2023 ?sales2023 ; ex:sales2024 ?sales2024 ; ex:sales2025 ?sales2025 .
  BIND(REPLACE(STR(?company), "http://example.org/company/", "") AS ?companyName)
}
GROUP BY ?companyName ORDER BY DESC(?totalSales)
```

The other queries cover bidirectional competitor pairs, electric models ranked by horsepower, SUV comparisons, the ownership hierarchy, fuel-type analysis and per-model performance.

## Reproducing it

1. Start GraphDB (free edition) and create a repository.
2. Import `OWL_ontology.ttl`.
3. Generate instance triples from the CSV with the RMLMapper (`java -jar rmlmapper.jar -m rml_mapping.ttl -o data/triples.ttl`) or import the pre-generated `data/triples.ttl` and `data/competitors.ttl`.
4. Import `shacl.ttl` as a SHACL shapes graph; the repository reports conformance on load.
5. Paste any query from `sparql.ttl` into the SPARQL editor.

## Why it is modelled this way

The automotive domain is a good fit for a graph: ownership is hierarchical and sometimes multi-level, competition is many-to-many and symmetric, and specifications are a bundle of typed facts that change per model year. Those are exactly the relationships that are awkward in a relational schema and natural as OWL property characteristics plus SPARQL property paths. SHACL is used separately from the ontology so that data-quality rules (every car needs a manufacturer and a sales record) can be tightened without changing the semantics.
