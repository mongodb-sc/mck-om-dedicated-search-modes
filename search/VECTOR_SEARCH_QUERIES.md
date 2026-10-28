# Vector Search Example Queries

Example `$vectorSearch` queries for the `sample_mflix.embedded_movies` collection. Run them in `mongosh` connected to the primary.

## Current index

```javascript
// sample_mflix.embedded_movies -> "vector_index"
{
  fields: [
    {
      type: "vector",
      path: "plot_embedding_voyage_3_large",
      numDimensions: 2048,
      similarity: "dotProduct",
      quantization: "scalar"
    }
  ]
}
```

| # | Feature | Works with current index? |
|---|---------|---------------------------|
| 1 | Basic ANN search | Yes |
| 2 | Returning the similarity score | Yes |
| 3 | ENN (exact) search | Yes |
| 4 | Post-filtering with `$match` | Yes |
| 5 | Post-processing (`$group`, `$lookup`, ...) | Yes |
| 6 | Query plan with `explain` | Yes |
| 7 | Pre-filtering (`filter`) | No, needs `filter` fields |
| 8 | Stored source (`returnStoredSource`) | No, needs `storedSource` |
| 9 | Quantization options | No, needs a new index |
| 10 | Indexing method (`flat` / `hnsw`) | No, needs a new index |
| 11 | Similarity functions | No, needs a new index |
| 12 | Automated embedding (`autoEmbed`) | No, needs `auto-embeddings.yaml` and a new index |
| 13 | Nested vectors (`nestedRoot`) | No, needs `sample_airbnb` and a new index |
| 14 | Hybrid search (`$rankFusion`) | No, needs a `search` index on `embedded_movies` |

## Setup: query vector

`numDimensions` is 2048, so the examples reuse the embedding of an existing document as the query vector instead of pasting it inline.

```javascript
use sample_mflix

const src = db.embedded_movies.findOne(
  { plot_embedding_voyage_3_large: { $exists: true } },
  { title: 1, plot: 1, genres: 1, year: 1, plot_embedding_voyage_3_large: 1 }
);
const qv = src.plot_embedding_voyage_3_large;
print(src.title, "|", src.plot);
```

The query vector must have the same model and number of dimensions as the indexed field (`voyage-3-large`, 2048).

---

## 1. Basic ANN search

Approximate Nearest Neighbor search. `numCandidates` is the number of nearest neighbors considered, and `limit` is the number of documents returned. `limit` must be less than or equal to `numCandidates`. A common starting point is `numCandidates` = 20 × `limit`.

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 100,
      limit: 5
    }
  },
  { $project: { _id: 0, title: 1, plot: 1 } }
]);
```

**Expected result:** 5 documents. The first is `src` itself, followed by movies whose plots are semantically similar to `src.plot`.

## 2. Returning the similarity score

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 100,
      limit: 5
    }
  },
  { $project: { _id: 0, title: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** The same 5 documents as query 1, each with a `score` field, sorted by descending score. With `dotProduct` the score is normalized to `[0, 1]` as `(1 + dotProduct) / 2`, so the first result (`src` itself) has a score of about `1.0`.

## 3. ENN (exact) search

`exact: true` runs an Exact Nearest Neighbor search that compares the query vector with every indexed vector. `numCandidates` must be omitted.

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      exact: true,
      limit: 5
    }
  },
  { $project: { _id: 0, title: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** The true top 5 matches. Compare them with query 2 to measure ANN accuracy (recall). They are usually the same or nearly the same. If ANN misses results, increase `numCandidates`. ENN is slower than ANN on large collections.

## 4. Post-filtering with `$match`

Without `filter` fields in the index, you can only filter after `$vectorSearch`. This can return fewer documents than `limit`.

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 200,
      limit: 20
    }
  },
  { $match: { year: { $gte: 2000 } } },
  { $limit: 5 },
  { $project: { _id: 0, title: 1, year: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** Up to 5 documents with `year >= 2000` taken from the top 20 semantic matches. If fewer than 5 of the top 20 match, fewer than 5 documents are returned. Query 7 shows how pre-filtering avoids this.


## 5. Post-processing (`$group`, `$lookup`, ...)

`$vectorSearch` must be the first stage of the pipeline. Any aggregation stage can follow it.

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 200,
      limit: 20
    }
  },
  { $unwind: "$genres" },
  { $group: { _id: "$genres", count: { $sum: 1 }, avgScore: { $avg: { $meta: "vectorSearchScore" } } } },
  { $sort: { count: -1 } }
]);
```

**Expected result:** One document per genre found in the top 20 matches, with the number of movies and the average score for each. Genres from `src.genres` should rank near the top.

## 6. Query plan with `explain`

```javascript
db.embedded_movies.explain("executionStats").aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 100,
      limit: 5
    }
  }
]);
```

**Expected result:** A plan that includes a `$vectorSearch` stage, the `mongot` query details, and execution statistics (timings and the number of documents returned).

---

## 7. Pre-filtering (`filter`)

**Index required:** add `filter` fields to `vector_index`. Wait until `db.embedded_movies.getSearchIndexes("vector_index")` shows `status: "READY"`.

```javascript
db.embedded_movies.updateSearchIndex("vector_index", {
  fields: [
    { type: "vector", path: "plot_embedding_voyage_3_large", numDimensions: 2048, similarity: "dotProduct", quantization: "scalar" },
    { type: "filter", path: "genres" },
    { type: "filter", path: "year" },
    { type: "filter", path: "rated" }
  ]
});
```

Filtering happens before the vector search, so `limit` documents are returned whenever enough documents match. Supported operators: `$eq`, `$ne`, `$gt`, `$lt`, `$gte`, `$lte`, `$in`, `$nin`, `$not`, `$nor`, `$and`, `$or`.

**7a. Comparison (`$gte`, `$lt`)**

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 200,
      limit: 5,
      filter: { year: { $gte: 1990, $lt: 2000 } }
    }
  },
  { $project: { _id: 0, title: 1, year: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** 5 movies from the 1990s that are the most similar to `src`.

**7b. Array membership and logical operators (`$in`, `$nin`, `$and`, `$or`)**

```javascript
filter: {
  $and: [
    { genres: { $in: ["Action", "Adventure"] } },
    { genres: { $nin: ["Animation"] } },
    { $or: [ { rated: "PG-13" }, { year: { $gte: 2010 } } ] }
  ]
}
```

**Expected result:** Only Action or Adventure movies that are not Animation and that are rated `PG-13` or released in 2010 or later. Results are sorted by similarity.

**7c. Negation (`$ne`, `$not`, `$nor`)**

```javascript
filter: { $nor: [ { rated: "R" }, { year: { $not: { $gte: 1980 } } } ] }
```

**Expected result:** Movies that are not rated `R` and were released in 1980 or later.

A `filter` on a field that the index doesn't define as a `filter` field fails with an error saying the path must be indexed as `filter`.

## 8. Stored source (`returnStoredSource`)

**Index required:** add `storedSource` so that `mongot` stores the listed fields. The query then returns them directly, without looking up the full documents in `mongod`.

```javascript
db.embedded_movies.updateSearchIndex("vector_index", {
  fields: [
    { type: "vector", path: "plot_embedding_voyage_3_large", numDimensions: 2048, similarity: "dotProduct", quantization: "scalar" },
    { type: "filter", path: "genres" },
    { type: "filter", path: "year" }
  ],
  storedSource: { include: ["title", "plot", "genres", "year"] }
});
```

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 100,
      limit: 5,
      returnStoredSource: true
    }
  },
  { $project: { title: 1, year: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** The same movies as query 2. Each document contains only `_id` and the stored fields (`title`, `plot`, `genres`, `year`). Fields that aren't stored, such as `cast` and `plot_embedding_voyage_3_large`, are not returned.

---

## 9. Quantization options

**Index required:** a second index on the same field with a different `quantization` value. Values: `none` (full-fidelity `float32`), `scalar` (current index, `int8`), `binary` (`int1`).

```javascript
db.embedded_movies.createSearchIndex("vector_index_binary", "vectorSearch", {
  fields: [
    { type: "vector", path: "plot_embedding_voyage_3_large", numDimensions: 2048, similarity: "dotProduct", quantization: "binary" }
  ]
});
```

```javascript
db.embedded_movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index_binary",
      path: "plot_embedding_voyage_3_large",
      queryVector: qv,
      numCandidates: 200,
      limit: 5
    }
  },
  { $project: { _id: 0, title: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** Mostly the same movies as query 2. `binary` uses the least memory but has lower accuracy, so the order can differ slightly. Raise `numCandidates` to improve recall.

## 10. Indexing method (`flat` / `hnsw`)

**Index required:** `indexingMethod: "flat"` stores vectors without a graph, so every query scans all vectors. `hnsw` (default) builds a graph that you can tune with `hnswOptions`.

```javascript
db.embedded_movies.createSearchIndex("vector_index_flat", "vectorSearch", {
  fields: [
    { type: "vector", path: "plot_embedding_voyage_3_large", numDimensions: 2048, similarity: "dotProduct", indexingMethod: "flat" }
  ]
});

db.embedded_movies.createSearchIndex("vector_index_hnsw", "vectorSearch", {
  fields: [
    {
      type: "vector", path: "plot_embedding_voyage_3_large", numDimensions: 2048, similarity: "dotProduct",
      indexingMethod: "hnsw",
      hnswOptions: { maxEdges: 32, numEdgeCandidates: 200 }
    }
  ]
});
```

Run query 2 with `index: "vector_index_flat"`, then with `index: "vector_index_hnsw"`.

**Expected result:** `vector_index_flat` returns the same results as the ENN query (query 3). `vector_index_hnsw` returns ANN results, which are usually the same. Higher `maxEdges` and `numEdgeCandidates` improve recall but use more memory and take longer to build.

## 11. Similarity functions

**Index required:** an index with a different `similarity` value. Values: `euclidean`, `cosine`, `dotProduct` (current index).

```javascript
db.embedded_movies.createSearchIndex("vector_index_euclidean", "vectorSearch", {
  fields: [
    { type: "vector", path: "plot_embedding_voyage_3_large", numDimensions: 2048, similarity: "euclidean", quantization: "scalar" }
  ]
});
```

Run query 2 with `index: "vector_index_euclidean"`.

**Expected result:** The ranking is the same as query 2, because Voyage AI embeddings are normalized to length 1. The scores differ: `euclidean` is normalized as `1 / (1 + distance)`.

## 12. Automated embedding (`autoEmbed`)

**Prerequisite:** deploy `auto-embeddings.yaml` so that `mongot` can call the Voyage AI API.

**Index required:** an `autoEmbed` field. `mongot` generates and stores the embeddings for `fullplot`, so documents don't need a vector field.

```javascript
db.movies.createSearchIndex("autoembed_index", "vectorSearch", {
  fields: [
    { type: "autoEmbed", modality: "text", path: "fullplot", model: "voyage-4" },
    { type: "filter", path: "genres" },
    { type: "filter", path: "year" }
  ]
});
```

Use `query.text` instead of `queryVector`. The two are mutually exclusive.

```javascript
db.movies.aggregate([
  {
    $vectorSearch: {
      index: "autoembed_index",
      path: "fullplot",
      query: { text: "young heroes caught in a war between rival kingdoms" },
      numCandidates: 100,
      limit: 5,
      filter: { year: { $gte: 1990 } }
    }
  },
  { $project: { _id: 0, title: 1, year: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** 5 movies from 1990 or later whose `fullplot` is semantically close to the query text, even if they don't contain its words.

## 13. Nested vectors (`nestedRoot`)

**Prerequisite:** import the `sample_airbnb` database the same way as `sample_mflix` in `README.md`, and deploy `auto-embeddings.yaml`.

**Index required:** `nestedRoot` names the array (`reviews`) whose elements are indexed as separate child documents.

```javascript
use sample_airbnb

db.listingsAndReviews.createSearchIndex("autoembed_index", "vectorSearch", {
  fields: [
    { type: "autoEmbed", modality: "text", path: "reviews.comments", model: "voyage-4", similarity: "cosine", indexingMethod: "hnsw" },
    { type: "filter", path: "address.country" },
    { type: "filter", path: "bedrooms" },
    { type: "filter", path: "property_type" },
    { type: "filter", path: "reviews.date" }
  ],
  nestedRoot: "reviews"
});
```

`filter` applies to fields of the child documents, and `parentFilter` applies to root-level fields. When both are set, they are combined with `AND`. `nestedOptions.scoreMode` sets how child scores become the parent score: `max` (default) or `avg`.

```javascript
db.listingsAndReviews.aggregate([
  {
    $vectorSearch: {
      index: "autoembed_index",
      path: "reviews.comments",
      query: { text: "quiet place close to the beach" },
      numCandidates: 100,
      limit: 5,
      filter: { "reviews.date": { $gte: ISODate("2018-01-01") } },
      parentFilter: { $and: [ { "address.country": "Portugal" }, { bedrooms: { $gte: 2 } } ] },
      nestedOptions: { scoreMode: "avg" }
    }
  },
  { $project: { _id: 0, name: 1, "address.country": 1, bedrooms: 1, score: { $meta: "vectorSearchScore" } } }
]);
```

**Expected result:** Up to 5 listings in Portugal with 2 or more bedrooms, ranked by the average similarity of their reviews from 2018 or later. Each listing appears once, even if several reviews match. With `scoreMode: "max"`, a listing with a single highly relevant review ranks higher.

## 14. Hybrid search (`$rankFusion`)

**Index required:** a `search` index on `embedded_movies`, in addition to `vector_index`. `$rankFusion` requires MongoDB 8.1 or later.

```javascript
db.embedded_movies.createSearchIndex("text_index", {
  mappings: { dynamic: false, fields: { title: { type: "string" }, plot: { type: "string" } } }
});
```

`$rankFusion` runs both pipelines and merges the results with Reciprocal Rank Fusion. `weights` sets how much each pipeline contributes.

```javascript
db.embedded_movies.aggregate([
  {
    $rankFusion: {
      input: {
        pipelines: {
          vectorPipeline: [
            { $vectorSearch: { index: "vector_index", path: "plot_embedding_voyage_3_large", queryVector: qv, numCandidates: 100, limit: 20 } }
          ],
          fullTextPipeline: [
            { $search: { index: "text_index", text: { query: "space", path: ["title", "plot"] } } },
            { $limit: 20 }
          ]
        }
      },
      combination: { weights: { vectorPipeline: 0.7, fullTextPipeline: 0.3 } },
      scoreDetails: true
    }
  },
  { $limit: 5 },
  { $project: { _id: 0, title: 1, scoreDetails: { $meta: "scoreDetails" } } }
]);
```

**Expected result:** 5 movies ranked by the combined score. Movies that appear in both pipelines rank highest. `scoreDetails` shows each pipeline's rank and weight for every result.
