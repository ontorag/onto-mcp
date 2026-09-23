# OntoRAG ontology catalog

The catalog of **baseline ontologies** that OntoRAG pipelines align to before inducing anything new. It serves the catalog two ways: as a REST API and as an MCP server, both from one FastAPI app.

- Live at **https://mcp.rpg-schema.org**: REST at the root, MCP at `/mcp`
- More about the catalog's role: https://ontorag.org/tools/#catalog

## Catalog contents

Each entry lives in [`data/ontologies/catalog.json`](data/ontologies/catalog.json) and points to a Turtle file in the same folder.

| Slug | File | Contents |
|---|---|---|
| `rpg` | `rpg.ttl` | rpg-schema, the tabletop RPG domain ontology (`http://www.rpg-schema.org/1.0/`) |
| `foaf` | `foaf.ttl` | FOAF |
| `dublincore` | `dc.ttl` | Dublin Core terms |
| `schemaorg` | `schemaorg.ttl` | schema.org |
| `fitd` | `fitd.ttl` | Forged in the Dark SRD |
| `cbrpnk` | `cbrpnk.ttl` | CBR+PNK SRD |
| `gumshoe` | `gumshoe_srd.ttl` | GUMSHOE SRD |

## REST API

| Method | Path | Returns |
|---|---|---|
| GET | `/` | Service status and the number of registered ontologies |
| GET | `/health` | `{"ok": true}` |
| GET | `/ontologies` | Every entry: slug, label, namespace, description, tags |
| GET | `/ontologies/{slug}` | One ontology as an OntoRAG schema card, with class and property counts |
| GET | `/search/classes?q=` | Classes whose name or description contains `q`, across all ontologies |
| GET | `/search/properties?q=` | Datatype and object properties matching `q` by name, domain, range or description |
| POST | `/compose` | `{"slugs": [...], "target_namespace": ""}` merged into one schema card |

## MCP tools

The MCP server is mounted at `/mcp` (streamable HTTP) and offers these tools:

- `list_ontologies`
- `inspect_ontology(slug)`
- `search_classes(query)`
- `search_properties(query)`
- `compose(slugs, target_namespace)`
- `add_ontology(slug, ttl_content, label, description, tags)`

```sh
claude mcp add --transport http ontorag-catalog https://mcp.rpg-schema.org/mcp
```

The REST route for adding an ontology is disabled in `app.py`. The MCP tool `add_ontology` writes to the catalog directory of the running instance, which a serverless deployment does not persist.

## Use from the ontorag CLI

`ontorag init-schema-card` composes baselines into the initial schema card of a new dataset. It looks for each slug in the local catalog (`--catalog`, default `./data/ontologies`) first. If the slug isn't there, it falls back to this service and calls `GET /ontologies/{slug}` on the base URL taken from `--mcp-url`, or `ONTORAG_MCP_URL`, or the default `https://mcp.rpg-schema.org/mcp` (the trailing `/mcp` is stripped).

```sh
ontorag init-schema-card --baselines rpg,schemaorg,dublincore --out ontology/schema_card.json
```

Every class and property in the resulting card records the baseline it came from in its `origin` field.

## Add an ontology

1. Put the Turtle file in `data/ontologies/`.
2. Add an entry to `data/ontologies/catalog.json` with `slug`, `label`, `namespace` (the prefix used in the file), `description`, `path` and `tags`.
3. Commit and redeploy.

Alternatively, `ontorag register-ontology` copies a TTL file into a local catalog and writes the entry for you.

## Run locally

```sh
uv sync
uv run uvicorn app:app --reload        # REST on http://127.0.0.1:8000, MCP on /mcp
```

Environment variables:

- `ONTORAG_CATALOG_DIR`: catalog folder (default `./data/ontologies`)
- `ONTORAG_VERBOSITY`: log level (0 to 2)

## Deploy

The app is deployed on Vercel, which serves the module-level `app` in `app.py`. The distribution name is `ontorag-catalog`. It was previously `ontorag-mcp`, which clashed with the [ontorag-mcp](https://github.com/ontorag/ontorag-mcp) dataset server.

## Vendored engine

The `ontorag/` folder is a vendored copy of an older version of the OntoRAG engine: the catalog, schema card and MCP modules this app imports. The maintained engine is the [`ontorag`](https://github.com/ontorag/ontorag) package on [PyPI](https://pypi.org/project/ontorag/), which also has `ontorag ontology-mcp` to serve a catalog locally.
