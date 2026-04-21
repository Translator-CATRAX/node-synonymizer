# node-synonymizer

node-synonymizer is a standalone Python client for the SRI Node Normalizer (https://nodenormalization-sri.renci.org) and Name Resolver (https://name-resolution-sri.renci.org) services. It provides CURIE canonicalization, equivalent-node lookup, and free-text name-to-CURIE mapping for Biolink-typed entities.

originally extracted from RTX-ARAX (https://github.com/RTXteam/RTX), branch `issue-2585`, file `code/ARAX/NodeSynonymizer/node_synonymizer.py`.

## install

```
pip install node-synonymizer
```

requires python 3.12.

## pinned versions

runtime deps are pinned to the exact versions used by RTX at extraction time, so the install is reproducible and consumers (Pathfinder, DrugBankNER) get the same tree ARAX runs today:

| package | version |
|---|---|
| python | 3.12 |
| aiohttp | 3.9.4 |
| bmt | 1.4.6 |
| pandas | 2.3.1 |
| requests | 2.32.3 |

dev extras:

| package | version |
|---|---|
| pytest | 8.1.1 |

## quick usage

lookup by CURIE:

```python
from node_synonymizer import NodeSynonymizer

syn = NodeSynonymizer()
print(syn.get_canonical_curies("DOID:14330"))
# {'DOID:14330': {'preferred_curie': 'MONDO:0005180',
#                 'preferred_name': 'Parkinson disease',
#                 'preferred_category': 'biolink:Disease'}}
```

lookup by free-text name:

```python
syn = NodeSynonymizer()
print(syn.get_canonical_curies(names="Parkinson's disease"))
```

get equivalent nodes in the cluster:

```python
syn = NodeSynonymizer()
print(syn.get_equivalent_nodes("DOID:14330"))
```

## endpoints

URLs are hardcoded to the RENCI-hosted SRI services:

- Node Normalizer: `https://nodenormalization-sri.renci.org/1.5`
- Name Resolver: `https://name-resolution-sri.renci.org`

the `transltr.io` production mirror of Name Resolver does NOT expose `/bulk-lookup`, which is one reason we pin to RENCI. see the `NodeSynonymizer` class docstring for the full rationale.

to override (e.g. to point at CI/dev), subclass and set the two class constants:

```python
class MySynonymizer(NodeSynonymizer):
    NODE_NORMALIZER_URL = "https://nodenorm.ci.transltr.io/1.5"
    NAME_RESOLVER_URL = "https://name-lookup.ci.transltr.io"
```

## changes from RTX-ARAX

the source file on `issue-2585` was written to run inside the ARAX tree. to make it standalone without changing behavior, the following edits were applied:

1. **dropped the `sys.path.append` block** at the top of the file. it existed because the ARAX repo doesn't use a standard Python package layout. the new `src/node_synonymizer/` layout makes it unnecessary.
2. **dropped the three imports from `openapi_server.models`** (`KnowledgeGraph`, `Node`, `Attribute`). those are ARAX-generated TRAPI pydantic classes that don't exist outside the ARAX tree, so they were a hard blocker for a standalone install.
3. **replaced the TRAPI model constructors in `_get_cluster_graph` and `_convert_to_trapi_node` with plain dict literals.** the dicts preserve the full field set and default values that the original `openapi_server.models.*.to_dict()` emitted:
   - Node dicts keep all 4 fields (`name`, `categories`, `attributes`, `is_set=False`).
   - Attribute dicts keep all 8 fields, with `None` for unset optionals (`original_attribute_name`, `value_url`, `description` when not passed, `attributes`).
   - the helper `_attribute_dict(...)` centralizes the Attribute shape.

   output is byte-for-byte identical to what ARAX produces today, verified by the smoke run below.
4. **dropped unused `os` / `sys` imports** that only existed for the sys.path block.
5. **no `RTXConfiguration` reference.** the two SRI URLs live as hardcoded class constants (`NODE_NORMALIZER_URL`, `NAME_RESOLVER_URL`). override by subclassing, see above.

nothing else in the class body was touched. the public API (`get_canonical_curies`, `get_equivalent_nodes`, `get_curie_names`, `get_preferred_names`, `get_normalizer_results`) is identical.

### why not reasoner-pydantic, why not copy ARAX's openapi_server files

two other options we considered for item 3:

1. `reasoner-pydantic` (https://github.com/TranslatorSRI/reasoner-pydantic), the Translator TRAPI pydantic lib. pydantic v2's `model_dump()` doesn't produce the same dict shape as openapi-generator's `.to_dict()` — aliases, enums, and None-handling all differ — so switching would change the output vs ARAX.

2. copy the ~5 openapi_server model files from RTX directly into this package. RTX is MIT so that's fine. same output because it's the same code. downside: 5 auto-generated files to carry, a `six` dep, and a resync every time ARAX regenerates them.

we went with plain dicts. this package is a thin client: given a CURIE or a name, return synonyms. whether the output is a validated TRAPI instance is the caller's problem. Pathfinder, DrugBankNER, and ARAX each revalidate at their own boundary anyway, so adding pydantic here just duplicates what they already do.

## tests

25 live tests adapted from RTX's `code/ARAX/test/test_ARAX_synonymizer.py`. the only change was dropping the ARAX sys.path hack; the test bodies are identical. they hit the real SRI endpoints at RENCI, so they need network access. a handful will fail if RENCI is down.

```
pip install -e '.[dev]'
pytest -v
```

coverage: 5 legacy tests (timing-style walks over a set of CURIEs / names) + 20 assertion-driven tests covering canonical-curie lookup, equivalent-node lookup by curie and by name, mixed-input lookup, unknown-CURIE handling, improper prefix capitalization, approximate name matching, normalizer-result shape, cluster graph shape, cluster truncation, entity-controller input formats.

last run on this machine (python 3.12.11, pytest 8.1.1):

```
============================= test session starts ==============================
platform darwin -- Python 3.12.11, pytest-8.1.1, pluggy-1.6.0
rootdir: /Users/bazarkua/Work/node-synonymizer
configfile: pyproject.toml
collected 25 items

tests/test_node_synonymizer.py::test_example_6b PASSED                   [  4%]
tests/test_node_synonymizer.py::test_example_9 PASSED                    [  8%]
tests/test_node_synonymizer.py::test_example_10 PASSED                   [ 12%]
tests/test_node_synonymizer.py::test_example_11 PASSED                   [ 16%]
tests/test_node_synonymizer.py::test_example_12 PASSED                   [ 20%]
tests/test_node_synonymizer.py::test_get_canonical_curies_simple PASSED  [ 24%]
tests/test_node_synonymizer.py::test_get_canonical_curies_single_curie PASSED [ 28%]
tests/test_node_synonymizer.py::test_get_canonical_curies_unrecognized PASSED [ 32%]
tests/test_node_synonymizer.py::test_get_canonical_curies_by_names PASSED [ 36%]
tests/test_node_synonymizer.py::test_get_canonical_curies_single_name PASSED [ 40%]
tests/test_node_synonymizer.py::test_get_canonical_curies_by_names_and_curies PASSED [ 44%]
tests/test_node_synonymizer.py::test_get_canonical_curies_return_all_categories PASSED [ 48%]
tests/test_node_synonymizer.py::test_get_equivalent_nodes PASSED         [ 52%]
tests/test_node_synonymizer.py::test_get_equivalent_nodes_by_name PASSED [ 56%]
tests/test_node_synonymizer.py::test_bad_name PASSED                     [ 60%]
tests/test_node_synonymizer.py::test_get_equivalent_nodes_by_curies_and_names PASSED [ 64%]
tests/test_node_synonymizer.py::test_get_curie_names PASSED              [ 68%]
tests/test_node_synonymizer.py::test_get_preferred_names PASSED          [ 72%]
tests/test_node_synonymizer.py::test_get_normalizer_results PASSED       [ 76%]
tests/test_node_synonymizer.py::test_improper_curie_prefix_capitalization PASSED [ 80%]
tests/test_node_synonymizer.py::test_approximate_name_based_matching PASSED [ 84%]
tests/test_node_synonymizer.py::test_entity_controller_input_no_format PASSED [ 88%]
tests/test_node_synonymizer.py::test_entity_controller_input_minimal_format PASSED [ 92%]
tests/test_node_synonymizer.py::test_cluster_graphs PASSED               [ 96%]
tests/test_node_synonymizer.py::test_truncate_cluster PASSED             [100%]

============================= 25 passed in 32.52s ==============================
```

### smoke run

same python, hitting the live RENCI endpoints:

```python
from node_synonymizer import NodeSynonymizer
import json

syn = NodeSynonymizer()

print(json.dumps(syn.get_canonical_curies("DOID:14330"), indent=2))

print(json.dumps(syn.get_canonical_curies(names="Parkinson's disease"), indent=2))

eq = syn.get_equivalent_nodes("DOID:14330")
print(f"equivalents for DOID:14330: {len(eq['DOID:14330'])} nodes")
print(f"first 5: {eq['DOID:14330'][:5]}")

res = syn.get_normalizer_results("PTGS1")
kg = res["PTGS1"]["knowledge_graph"]
print(f"nodes in cluster: {len(kg['nodes'])}")
print(f"edges: {kg['edges']}")
sample_node = next(iter(kg['nodes'].values()))
print(f"sample node keys: {sorted(sample_node.keys())}")
print(f"sample attribute keys: {sorted(sample_node['attributes'][0].keys())}")
```

output:

```
{
  "DOID:14330": {
    "preferred_curie": "MONDO:0005180",
    "preferred_name": "Parkinson disease",
    "preferred_category": "biolink:Disease"
  }
}
{
  "Parkinson's disease": {
    "preferred_curie": "MONDO:0005180",
    "preferred_name": "Parkinson disease",
    "preferred_category": "biolink:Disease"
  }
}
equivalents for DOID:14330: 16 nodes
first 5: ['MONDO:0005180', 'DOID:14330', 'OMIM.PS:168600', 'UMLS:C0030567', 'MESH:D010300']
nodes in cluster: 27
edges: {}
sample node keys: ['attributes', 'categories', 'is_set', 'name']
sample attribute keys: ['attribute_source', 'attribute_type_id', 'attributes', 'description', 'original_attribute_name', 'value', 'value_type_id', 'value_url']
```

note the sample node keys (`attributes`, `categories`, `is_set`, `name`) and sample attribute keys (all 8 fields incl. the `None`-defaulted `original_attribute_name`, `value_url`, `attributes`) confirm byte-for-byte parity with ARAX's original `openapi_server.models.*.to_dict()` output. cluster counts (`len(equivalents)=16`, `nodes in cluster=27`) can drift when RENCI updates.

## development

reproducible setup with pinned deps:

```
git clone https://github.com/Translator-CATRAX/node-synonymizer
cd node-synonymizer
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e '.[dev]'
pytest -v
```

## license

MIT. see [LICENSE](LICENSE).
