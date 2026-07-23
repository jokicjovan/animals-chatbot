# Animals Chatbot

Ask a question about animals in plain language — *"Where do lions live?"*, *"What does an alpaca
eat?"* — and get a plain-language answer back.

Behind the scenes the question is translated into a formal database query by a language model,
answered from a structured knowledge graph of animal facts, and turned back into a sentence. The
project is intended as an educational tool for preschool and school-aged children who want to explore
the animal kingdom in an accessible and engaging way.

**Stack:** FastAPI · LangChain · OpenAI GPT-4o · Virtuoso · RDF/SPARQL · rdflib · Jinja2

![System Architecture](assets/system-architecture.png)

## How it works

Rather than storing animal facts in ordinary tables, the project stores them as a **knowledge graph**:
a web of connected statements such as *lion → belongs to → Felidae* or *lion → lives in → Africa*.
This structure lets the system answer questions that span several relationships at once, and makes the
data meaningful to a machine rather than just readable.

The chain that answers a question runs in six steps:

1. **Ontology loading** — at startup the app runs a SPARQL `CONSTRUCT` query against the Virtuoso
   endpoint via SPARQLWrapper, writes the result to a temporary RDF file, and loads it into an
   in-memory graph through LangChain's `RdfGraph` interface.
2. **User query** — a free-form question is submitted to the backend over HTTP POST.
3. **SPARQL generation** — GPT-4o receives the ontology schema and a structured prompt template, and
   returns a `SELECT` query matching the user's intent and the graph's structure.
4. **Query execution** — the generated query runs against Virtuoso and returns JSON results.
5. **Answer construction** — the original question and the raw results go back to GPT-4o, which
   turns them into a human-readable answer.
6. **Response** — the answer is returned to the frontend as JSON.

LangChain's `SparqlQAChain` is central to this architecture, acting as the bridge between the user's
natural language input and the RDF knowledge graph.

## Getting started

### Prerequisites

- Python 3.10+
- Docker (for Virtuoso)
- An [OpenAI API key](https://platform.openai.com/api-keys)
- An [API Ninjas](https://api-ninjas.com/api/animals) key, for populating the ontology

### 1. Start Virtuoso

```bash
docker compose up -d
```

Virtuoso runs at `http://localhost:8890`, with the SPARQL endpoint at `/sparql` and the Conductor
admin interface at `/conductor`.

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Populate the ontology

The base ontology ships empty, so it needs to be filled with animal data first.

```bash
cd ontology
cp .env.example .env        # then add your API Ninjas key
python populate_ontology.py
```

This writes `mega_populated_ontology.rdf`. Load that file into Virtuoso through the Conductor
interface (`http://localhost:8890/conductor` → **Linked Data → Quad Store Upload**), using the graph
IRI `http://www.semanticweb.org/tehno-trube/ontologies/2024/11/animals_ontology.owl`.

### 4. Run the app

The application reads `OPENAI_API_KEY` from a `.env` file in the `app` directory, and must be started
from that directory (static files, templates, and imports are resolved relative to it).

```bash
cd app
cp .env.example .env        # then add your OpenAI key
uvicorn main:app --reload
```

Open `http://localhost:8000`.

## Animal ontology

The ontology defines a structured representation of animal knowledge, modelling the biological and
ecological aspects of an animal: where it sits in the tree of life, what it is like, and where it
lives.

**Classes**

| Class | Purpose |
|---|---|
| `Animal` | An individual animal instance |
| `Characteristics` | Biological and ecological data about an animal |
| `Location` | Geographical habitat data |

**Taxonomic levels:** Kingdom → Phylum → Class → Order → Family → Genus → Scientific_name

**Object properties**

- `belongsTo` — links taxonomic entities across classification levels
- `hasScientificName` — connects an `Animal` to its `Scientific_name`
- `hasCharacteristics` — associates an `Animal` with its `Characteristics`
- `livesIn` — specifies the `Location` where an `Animal` lives

**Datatype properties** — attributes such as `diet`, `color`, `lifespan`, `gestation_period`,
`top_speed`, `prey`, and `skin_type`, all represented as `xsd:string` literals and attached mostly to
`Characteristics`.

This design exists to support semantic reasoning and precise querying: because the taxonomy is
expressed as explicit links rather than plain text, a question like *"which mammals are carnivores?"*
can be answered by following relationships through the graph.

<img src="./assets/visualization.svg">

## Populating the ontology

`ontology/populate_ontology.py` enriches the empty base ontology with biological data from the
[API Ninjas Animals API](https://api-ninjas.com/api/animals). For each animal in a predefined list of
100 interesting animals, it retrieves taxonomy, geographical locations, and characteristics, then
transforms that hierarchical JSON into RDF triples using `rdflib`.

The script:

1. Loads the base ontology (`empty_animals_ontology.rdf`) into an `rdflib.Graph`
2. Defines the custom namespace, so the RDF data is structured consistently
3. Creates URIs for each animal and its taxonomic ranks, adding `belongsTo` triples that mirror the
   biological hierarchy
4. Adds triples for scientific name, habitats, and other characteristics, linked to the animal entity
5. Sanitizes property keys (stripping special characters) so JSON attributes become valid URIs
6. Serializes the populated graph to `mega_populated_ontology.rdf` for querying and reasoning

## Frontend

A simple web interface served with Jinja2 templates, where users type natural-language questions about
animals and read the answers.

<img src="./assets/frontend1.jpg" width="400"> <img src="./assets/frontend2.jpg" width="400">

## Authors

- [Jovan Jokić](https://github.com/jokicjovan)
- [Bojan Mijanović](https://github.com/bmijanovic)
- [Vukašin Bogdanović](https://github.com/vukasinb7)
