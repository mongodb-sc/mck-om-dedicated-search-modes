# Text Search Example Queries

Example MongoDB Search queries for the `sample_mflix.movies` collection.

Run them in `mongosh` connected to the primary (see [README.md](README.md#create-a-search-index)):
```
use sample_mflix
```

Unless a section says otherwise, the queries use the existing `default` index:
```
db.movies.createSearchIndex("default", { mappings: { dynamic: true } });
```

> Expected results come from the MongoDB Search documentation for the same dataset. Scores can differ slightly between versions.

| # | Feature | Index |
|---|---------|-------|
| 1 | `text`: basic term search | `default` |
| 2 | `text`: fuzzy search (default options) | `default` |
| 3 | `text`: fuzzy search with `maxEdits` and `prefixLength` | `default` |
| 4 | `text`: `matchCriteria: "all"` | `default` |
| 4.1 | `text`: `matchCriteria: "all"` with stemming | `english_index` (see [Stemming](#41-text-matchcriteria-all-with-stemming)) |
| 5 | `phrase`: exact phrase | `default` |
| 6 | `phrase`: multiple phrases | `default` |
| 7 | `phrase`: `slop` | `default` |
| 8 | `$searchMeta`: count and number facet | `default` |
| 9 | `text` and `phrase` with synonyms | `default` with a synonym mapping (see [Synonyms](#9-synonyms)) |
| 10 | `autocomplete`: search as you type | `autocomplete_index` (see [Autocomplete](#10-autocomplete)) |
| 11 | `autocomplete` with a string facet | `autocomplete_index` |

---

## 1. `text`: basic term search
Searches the `title` field for the term `surfer`.
```
db.movies.aggregate([
  { $search: { text: { path: "title", query: "surfer" } } },
  { $project: { _id: 0, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result:
```
{ "title" : "Soul Surfer", "score" : 4.518949508666992 }
{ "title" : "Little Surfer Girl", "score" : 3.8856077194213867 }
{ "title" : "Fantastic 4: Rise of the Silver Surfer", "score" : 2.489800453186035 }
```

## 2. `text`: fuzzy search (default options)
Finds titles that match the misspelled query `naw yark`. The default options are `maxEdits: 2` and `maxExpansions: 50`, with `prefixLength` disabled.
```
db.movies.aggregate([
  { $search: { text: { path: "title", query: "naw yark", fuzzy: {} } } },
  { $limit: 10 },
  { $project: { _id: 0, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result:
```
{ "title" : "New York, New York", "score" : 3.6040148735046387 }
{ "title" : "New York", "score" : 3.323730945587158 }
{ "title" : "New York Stories", "score" : 2.8579015731811523 }
{ "title" : "New York Minute", "score" : 2.8579015731811523 }
{ "title" : "Synecdoche, New York", "score" : 2.8579015731811523 }
{ "title" : "New York Doll", "score" : 2.8579015731811523 }
{ "title" : "Little New York", "score" : 2.8579015731811523 }
{ "title" : "Escape from New York", "score" : 2.506596088409424 }
{ "title" : "Naked in New York", "score" : 2.506596088409424 }
{ "title" : "Autumn in New York", "score" : 2.506596088409424 }
```

## 3. `text`: fuzzy search with `maxEdits` and `prefixLength`
Allows one character edit per term. `prefixLength: 2` means the first 2 characters of each term can't change, so `naw` can't become `new`.
```
db.movies.aggregate([
  { $search: { text: { path: "title", query: "naw yark", fuzzy: { maxEdits: 1, prefixLength: 2 } } } },
  { $limit: 8 },
  { $project: { _id: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result:
```
{ "_id" : ObjectId("573a1396f29313caabce5646"), "title" : "The Longest Yard", "score" : 2.914206027984619 }
{ "_id" : ObjectId("573a13aff29313caabd31ed8"), "title" : "The Longest Yard", "score" : 2.914206027984619 }
{ "_id" : ObjectId("573a13b7f29313caabd4ad8b"), "title" : "Stomp the Yard", "score" : 2.914206027984619 }
{ "_id" : ObjectId("573a13eaf29313caabdcf410"), "title" : "Naz & Maalik", "score" : 2.5460386276245117 }
{ "_id" : ObjectId("573a1393f29313caabcddbed"), "title" : "La nao capitana", "score" : 2.1892051696777344 }
{ "_id" : ObjectId("573a1399f29313caabcee781"), "title" : "Kabhi Haan Kabhi Naa", "score" : 1.9200985431671143 }
{ "_id" : ObjectId("573a13a2f29313caabd0b815"), "title" : "Kaho Naa... Pyaar Hai", "score" : 1.9200985431671143 }
{ "_id" : ObjectId("573a13a7f29313caabd1b5c0"), "title" : "Oysters at Nam Kee's", "score" : 1.9200985431671143 }
```

## 4. `text`: `matchCriteria: "all"`
Returns only plots that contain both `automobile` and `race`, in any position.
```
db.movies.aggregate([
  { $search: { text: { path: "plot", query: "automobile race", matchCriteria: "all" } } },
  { $limit: 20 },
  { $project: { _id: 0, plot: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result:
```
[
  {
    plot: 'When a popular daredevil proposes an automobile race across three continents, his arch rival vows to beat him, while an ambitious female reporter has her own plans for victory.',
    title: 'The Great Race',
    score: 5.282209396362305
  }
]
```
Only "The Great Race" is returned. The `default` index analyzes `plot` with `lucene.standard`, which doesn't apply stemming, so `racing` doesn't match `race`. Query 4.1 shows how stemming changes this.

## 4.1 `text`: `matchCriteria: "all"` with stemming
> **The existing index doesn't support this.** Stemming needs `plot` to be analyzed with `lucene.english`.

Create `english_index`:
```
db.movies.createSearchIndex("english_index", {
  mappings: {
    dynamic: false,
    fields: { plot: { type: "string", analyzer: "lucene.english" } }
  }
});
```
Wait until `db.movies.getSearchIndexes("english_index")` shows `status: 'READY'`.

Run query 4 against `english_index`:
```
db.movies.aggregate([
  { $search: { index: "english_index", text: { path: "plot", query: "automobile race", matchCriteria: "all" } } },
  { $limit: 20 },
  { $project: { _id: 0, plot: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result: 3 documents. `lucene.english` reduces `racing` (including the `racing` in `drag-racing`) to `race`, so these plots also match. Scores differ from query 4 because stemming changes the term statistics used for scoring.
```
[
  {
    plot: 'A young driver, Speed Racer, aspires to be champion of the racing world with the help of his family and his high-tech Mach 5 automobile.',
    title: 'Speed Racer'
  },
  {
    plot: 'A gorgeous young automobile fanatic--and front to the hottest unsigned band on the West coast--finds herself caught up in illegal drag-racing competitions organized by exotic car fanatics.',
    title: 'Redline'
  },
  {
    plot: 'When a popular daredevil proposes an automobile race across three continents, his arch rival vows to beat him, while an ambitious female reporter has her own plans for victory.',
    title: 'The Great Race'
  }
]
```

## 5. `phrase`: exact phrase
Matches titles that contain `new york` as an ordered phrase. The default `slop` is `0`.
```
db.movies.aggregate([
  { $search: { phrase: { path: "title", query: "new york" } } },
  { $limit: 10 },
  { $project: { _id: 0, title: 1, score: { $meta: "searchScore" } } }
])
```

Expected result:
```
{ title: 'New York, New York', score: 6.786321640014648 }
{ title: 'New York', score: 6.258549213409424 }
{ title: 'New York Stories', score: 5.3813982009887695 }
{ title: 'New York Minute', score: 5.3813982009887695 }
{ title: 'Synecdoche, New York', score: 5.3813982009887695 }
{ title: 'New York Doll', score: 5.3813982009887695 }
{ title: 'Little New York', score: 5.3813982009887695 }
{ title: 'Escape from New York', score: 4.719893455505371 }
{ title: 'Naked in New York', score: 4.719893455505371 }
{ title: 'Autumn in New York', score: 4.719893455505371 }
```

## 6. `phrase`: multiple phrases
Matches titles that contain `the man` or `the moon`.
```
db.movies.aggregate([
  { $search: { phrase: { path: "title", query: ["the man", "the moon"] } } },
  { $limit: 10 },
  { $project: { _id: 0, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result:
```
{ title: 'The Man in the Moon', score: 4.4830474853515625 }
{ title: 'Shoot the Moon', score: 3.252699851989746 }
{ title: 'Kick the Moon', score: 3.252699851989746 }
{ title: 'The Man', score: 2.8923356533050537 }
{ title: 'The Moon and Sixpence', score: 2.8528637886047363 }
{ title: 'The Moon Is Blue', score: 2.8528637886047363 }
{ title: 'Racing with the Moon', score: 2.8528637886047363 }
{ title: 'Mountains of the Moon', score: 2.8528637886047363 }
{ title: 'Man on the Moon', score: 2.8528637886047363 }
{ title: 'Castaway on the Moon', score: 2.8528637886047363 }
```

## 7. `phrase`: `slop`
`slop: 5` lets the words `men` and `women` be up to 5 positions apart, in any order. Exact matches score higher.
```
db.movies.aggregate([
  { $search: { phrase: { path: "title", query: "men women", slop: 5 } } },
  { $project: { _id: 0, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result:
```
{ title: 'Men Without Women', score: 3.367523193359375 }
{ title: 'Men Vs Women', score: 3.367523193359375 }
{ title: 'Good Men, Good Women', score: 2.8529787063598633 }
{ title: 'The War Between Men and Women', score: 2.1851978302001953 }
{ title: 'Women Without Men', score: 1.9656763076782227 }
{ title: 'Women Vs Men', score: 1.9656763076782227 }
```

## 8. `$searchMeta`: count and number facet
Returns how many movies with `new york` in the title fall into each `year` bucket. It returns metadata only, not documents.
```
db.movies.aggregate([
  { $searchMeta: {
      facet: {
        operator: { phrase: { path: "title", query: "new york" } },
        facets: { yearFacet: { type: "number", path: "year", boundaries: [2000, 2005, 2010, 2015] } }
      }
  } }
])
```
Expected result:
```
[
  {
    count: { lowerBound: Long('27') },
    facet: {
      yearFacet: {
        buckets: [
          { _id: 2000, count: Long('3') },
          { _id: 2005, count: Long('8') },
          { _id: 2010, count: Long('4') }
        ]
      }
    }
  }
]
```

---

## 9. Synonyms
> **The existing index doesn't support this.** You need a synonym source collection and a synonym mapping in the `default` index.

**Step 1: Create the `synonymous_terms` collection**
```
db.synonymous_terms.insertMany([
  { mappingType: "equivalent", synonyms: ["car", "vehicle", "automobile"] },
  { mappingType: "equivalent", synonyms: ["attire", "dress", "apparel"] },
  { mappingType: "explicit", input: ["boat"], synonyms: ["boat", "vessel", "sail"] },
  { mappingType: "explicit", input: ["race"], synonyms: ["race", "rally", "contest"] }
])
```
- `equivalent`: every word in the list matches all the others.
- `explicit`: only the `input` word expands to the `synonyms`. The reverse doesn't apply.

**Step 2: Add the synonym mapping `my_synonyms` to the `default` index**
```
db.movies.updateSearchIndex("default", {
  mappings: { dynamic: true },
  synonyms: [ { name: "my_synonyms", analyzer: "lucene.standard", source: { collection: "synonymous_terms" } } ]
});
```
Wait until `db.movies.getSearchIndexes("default")` shows `status: 'READY'`.

### 9a. `text` with an `equivalent` mapping
```
db.movies.aggregate([
  { $search: { text: { path: "plot", query: "attire", synonyms: "my_synonyms", matchCriteria: "any" } } },
  { $limit: 5 },
  { $project: { _id: 0, plot: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result: plots that contain `attire`, `apparel` or `dress`. A query for `dress` returns similar documents.

### 9b. `text` with an `explicit` mapping
```
db.movies.aggregate([
  { $search: { text: { path: "plot", query: "boat race", synonyms: "my_synonyms", matchCriteria: "any" } } },
  { $limit: 10 },
  { $project: { _id: 0, plot: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result: plots that contain `boat`, `vessel` or `sail`, or `race`, `rally` or `contest`. A query for `vessel` or `rally` does **not** expand to the other words, because the mappings are explicit.

### 9c. `text` with synonyms and `matchCriteria: "all"`
```
db.movies.aggregate([
  { $search: { text: { path: "plot", query: "automobile race", matchCriteria: "all", synonyms: "my_synonyms" } } },
  { $limit: 20 },
  { $project: { _id: 0, plot: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result: plots that contain (`automobile` or `car` or `vehicle`) **and** (`race` or `rally` or `contest`). This returns more results than query #4.

### 9d. `phrase` with synonyms and `slop`
```
db.movies.aggregate([
  { $search: { phrase: { path: "plot", query: "automobile race", slop: 5, synonyms: "my_synonyms" } } },
  { $limit: 5 },
  { $project: { _id: 0, plot: 1, title: 1, score: { $meta: "searchScore" } } }
])
```
Expected result: plots with a word from `automobile`/`car`/`vehicle` up to 5 positions away from a word in `race`/`contest`/`rally`. `car race` returns similar results, but `automobile rally` doesn't match documents that contain `race`.

---

## 10. Autocomplete
> **The existing index doesn't support this.** `autocomplete` needs a field indexed as type `autocomplete`, and a string facet needs type `token`. Dynamic mappings create neither.

Create `autocomplete_index`:
```
db.movies.createSearchIndex("autocomplete_index", {
  mappings: {
    dynamic: false,
    fields: {
      title: [
        { type: "autocomplete", tokenization: "edgeGram", minGrams: 2, maxGrams: 15, foldDiacritics: false },
        { type: "token" }
      ]
    }
  }
});
```
`edgeGram` matches the start of each word. For other matching, use `tokenization: "rightEdgeGram"` to match the end of each word, or `"nGram"` to match any part of a word.

### 10a. Search as you type
```
db.movies.aggregate([
  { $search: { index: "autocomplete_index", autocomplete: { path: "title", query: "off" } } },
  { $limit: 10 },
  { $project: { _id: 0, title: 1 } }
])
```
Expected result with `edgeGram` (every title has a word that starts with `off`):
```
{ title: 'Off Beat' }
{ title: 'Off the Map' }
{ title: 'Off and Running' }
{ title: 'Hands off Mississippi' }
{ title: 'Taking Off' }
{ title: 'Noises Off...' }
{ title: 'Brassed Off' }
{ title: 'Face/Off' }
{ title: 'Benji: Off the Leash!' }
{ title: 'Set It Off' }
```
With `nGram`, the results also include titles where `off` is inside a word, such as `'Come Have Coffee with Us'`, `'An Officer and a Gentleman'` and `'Romanoff and Juliet'`.

## 11. `autocomplete` with a string facet
Groups movies whose title matches `Gravity` by unique title. This uses the `token` type on `title`.
```
db.movies.aggregate([
  { $searchMeta: {
      index: "autocomplete_index",
      facet: {
        operator: { autocomplete: { path: "title", query: "Gravity" } },
        facets: { titleFacet: { type: "string", path: "title" } }
      }
  } }
])
```
Expected result:
```
[{
   count: { lowerBound: Long("5") },
   facet: {
      titleFacet: {
         buckets: [
            { _id: 'Gravity', count: Long("3") },
            { _id: 'Defying Gravity', count: Long("1") },
            { _id: 'Laws of Gravity', count: Long("1") }
         ]
      }
   }
}]
```
