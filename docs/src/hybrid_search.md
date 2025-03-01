# Hybrid Search

LanceDB supports both semantic and keyword-based search. In real world applications, it is often useful to combine these two approaches to get the best best results. For example, you may want to search for a document that is semantically similar to a query document, but also contains a specific keyword. This is an example of *hybrid search*, a search algorithm that combines multiple search techniques.

## Hybrid search in LanceDB
You can perform hybrid search in LanceDB by combining the results of semantic and full-text search via a reranking algorithm of your choice. LanceDB provides multiple rerankers out of the box. However, you can always write a custom reranker if your use case need more sophisticated logic .

```python
import os

import lancedb
import openai
from lancedb.embeddings import get_registry
from lancedb.pydantic import LanceModel, Vector

db = lancedb.connect("~/.lancedb")

# Ingest embedding function in LanceDB table
# Configuring the environment variable OPENAI_API_KEY
if "OPENAI_API_KEY" not in os.environ:
# OR set the key here as a variable
    openai.api_key = "sk-..."
embeddings = get_registry().get("openai").create()

class Documents(LanceModel):
    vector: Vector(embeddings.ndims()) = embeddings.VectorField()
    text: str = embeddings.SourceField()

table = db.create_table("documents", schema=Documents)

data = [
    { "text": "rebel spaceships striking from a hidden base"},
    { "text": "have won their first victory against the evil Galactic Empire"},
    { "text": "during the battle rebel spies managed to steal secret plans"},
    { "text": "to the Empire's ultimate weapon the Death Star"}
]

# ingest docs with auto-vectorization
table.add(data)

# Create a fts index before the hybrid search
table.create_fts_index("text")
# hybrid search with default re-ranker
results = table.search("flower moon", query_type="hybrid").to_pandas()
```

By default, LanceDB uses `LinearCombinationReranker(weight=0.7)` to combine and rerank the results of semantic and full-text search. You can customize the hyperparameters as needed or write your own custom reranker. Here's how you can use any of the available rerankers:


### `rerank()` arguments
* `normalize`: `str`, default `"score"`:
    The method to normalize the scores. Can be "rank" or "score". If "rank", the scores are converted to ranks and then normalized. If "score", the scores are normalized directly.
* `reranker`: `Reranker`, default `LinearCombinationReranker(weight=0.7)`.
    The reranker to use. If not specified, the default reranker is used.


## Available Rerankers
LanceDB provides a number of re-rankers out of the box. You can use any of these re-rankers by passing them to the `rerank()` method. Here's a list of available re-rankers:

### Linear Combination Reranker
This is the default re-ranker used by LanceDB. It combines the results of semantic and full-text search using a linear combination of the scores. The weights for the linear combination can be specified. It defaults to 0.7, i.e, 70% weight for semantic search and 30% weight for full-text search.


```python
from lancedb.rerankers import LinearCombinationReranker

reranker = LinearCombinationReranker(weight=0.3) # Use 0.3 as the weight for vector search

results = table.search("rebel", query_type="hybrid").rerank(reranker=reranker).to_pandas()
```

Arguments
----------------
* `weight`: `float`, default `0.7`:
    The weight to use for the semantic search score. The weight for the full-text search score is `1 - weights`.
* `fill`: `float`, default `1.0`:
        The score to give to results that are only in one of the two result sets.This is treated as penalty, so a higher value means a lower score.
        TODO: We should just hardcode this-- its pretty confusing as we invert scores to calculate final score
* `return_score` : str, default `"relevance"`
        options are "relevance" or "all"
        The type of score to return. If "relevance", will return only the `_relevance_score. If "all", will return all scores from the vector and FTS search along with the relevance score.

### Cohere Reranker
This re-ranker uses the [Cohere](https://cohere.ai/) API to combine the results of semantic and full-text search. You can use this re-ranker by passing `CohereReranker()` to the `rerank()` method. Note that you'll need to set the `COHERE_API_KEY` environment variable to use this re-ranker.

```python
from lancedb.rerankers import CohereReranker

reranker = CohereReranker()

results = table.search("vampire weekend", query_type="hybrid").rerank(reranker=reranker).to_pandas()
```

Arguments
----------------
* `model_name`` : str, default `"rerank-english-v2.0"``
        The name of the cross encoder model to use. Available cohere models are:
        - rerank-english-v2.0
        - rerank-multilingual-v2.0
* `column` : str, default `"text"`
        The name of the column to use as input to the cross encoder model.
* `top_n` : str, default `None`
        The number of results to return. If None, will return all results.

!!! Note
    Only returns `_relevance_score`. Does not support `return_score = "all"`.

### Cross Encoder Reranker
This reranker uses the [Sentence Transformers](https://www.sbert.net/) library to combine the results of semantic and full-text search. You can use it by passing `CrossEncoderReranker()` to the `rerank()` method.

```python
from lancedb.rerankers import CrossEncoderReranker

reranker = CrossEncoderReranker()

results = table.search("harmony hall", query_type="hybrid").rerank(reranker=reranker).to_pandas()
```


Arguments
----------------
* `model` : str, default `"cross-encoder/ms-marco-TinyBERT-L-6"`
        The name of the cross encoder model to use. Available cross encoder models can be found [here](https://www.sbert.net/docs/pretrained_cross-encoders.html)
* `column` : str, default `"text"`
        The name of the column to use as input to the cross encoder model.
* `device` : str, default `None`
        The device to use for the cross encoder model. If None, will use "cuda" if available, otherwise "cpu".

!!! Note
    Only returns `_relevance_score`. Does not support `return_score = "all"`.


## Building Custom Rerankers
You can build your own custom reranker by subclassing the `Reranker` class and implementing the `rerank_hybrid()` method. Here's an example of a custom reranker that combines the results of semantic and full-text search using a linear combination of the scores.

The `Reranker` base interface comes with a `merge_results()` method that can be used to combine the results of semantic and full-text search. This is a vanilla merging algorithm that simply concatenates the results and removes the duplicates without taking the scores into consideration. It only keeps the first copy of the row encountered. This works well in cases that don't require the scores of semantic and full-text search to combine the results. If you want to use the scores or want to support `return_score="all"`, you'll need to implement your own merging algorithm.

```python

from lancedb.rerankers import Reranker
import pyarrow as pa

class MyReranker(Reranker):
    def __init__(self, param1, param2, ..., return_score="relevance"):
        super().__init__(return_score)
        self.param1 = param1
        self.param2 = param2
    
    def rerank_hybrid(self, vector_results: pa.Table, fts_results: pa.Table):
        # Use the built-in merging function
        combined_result = self.merge_results(vector_results, fts_results)
        
        # Do something with the combined results
        # ...

        # Return the combined results
        return combined_result

```

You can also accept additional arguments like a filter along with fts and vector search results

```python

from lancedb.rerankers import Reranker
import pyarrow as pa

class MyReranker(Reranker):
    ...
    
    def rerank_hybrid(self, vector_results: pa.Table, fts_results: pa.Table, filter: str):
        # Use the built-in merging function
        combined_result = self.merge_results(vector_results, fts_results)
        
        # Do something with the combined results & filter
        # ...

        # Return the combined results
        return combined_result

```
