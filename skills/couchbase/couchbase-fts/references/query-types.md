# FTS query types

All queries go through `cb_fts_search(index_name, query, size, from, highlight, fields, facets, cluster)`.

The `query` parameter is a JSON object whose shape determines the query type.

## Simple queries

### Match query (most common)

Tokenizes the search string and matches against a text field. Uses the field's analyzer.

```json
{ "match": "quick brown fox", "field": "description" }
```

For multi-word input, `match` performs an OR by default (any term matches). Use `operator: "and"` to require all terms:

```json
{ "match": "quick brown fox", "field": "description", "operator": "and" }
```

### Match phrase query

All terms must appear in order with no intervening terms.

```json
{ "match_phrase": "quick brown fox", "field": "description" }
```

Requires `include_term_vectors: true` on the field at index time.

### Term query

Exact match — no analysis applied. Use for keyword fields.

```json
{ "term": "active", "field": "status" }
```

### Prefix query

Matches terms starting with the prefix.

```json
{ "prefix": "prod", "field": "category" }
```

### Wildcard query

`*` = any sequence, `?` = single character. Applied to the indexed terms (post-analysis).

```json
{ "wildcard": "cou?hbase", "field": "product" }
```

### Fuzzy query

Matches terms within a given edit distance (Levenshtein). `fuzziness: 1` or `2`.

```json
{ "match": "cuchbase", "field": "product", "fuzziness": 1 }
```

### Regexp query

Matches terms matching a regular expression (BLEVE regexp syntax).

```json
{ "regexp": "cou.*ase", "field": "product" }
```

## Range queries

### Numeric range

```json
{ "min": 10.0, "max": 100.0, "inclusive_min": true, "inclusive_max": false, "field": "price" }
```

Omit `min` or `max` for open-ended ranges.

### Date range

```json
{
  "start": "2025-01-01T00:00:00Z",
  "end": "2025-12-31T23:59:59Z",
  "inclusive_start": true,
  "inclusive_end": false,
  "field": "created_at"
}
```

### Term range

Lexicographic range on keyword fields.

```json
{ "min": "apple", "max": "mango", "field": "product_name" }
```

## Compound queries

### Conjunction (AND)

All child queries must match.

```json
{
  "conjuncts": [
    { "match": "couchbase", "field": "description" },
    { "term": "active", "field": "status" }
  ]
}
```

### Disjunction (OR)

At least `min` child queries must match (default 1).

```json
{
  "disjuncts": [
    { "match": "couchbase", "field": "description" },
    { "match": "nosql", "field": "tags" }
  ],
  "min": 1
}
```

### Boolean query

Combines `must`, `should`, and `must_not` clauses.

```json
{
  "must":     { "conjuncts": [{ "term": "active", "field": "status" }] },
  "should":   { "disjuncts": [{ "match": "couchbase", "field": "description" }] },
  "must_not": { "disjuncts": [{ "term": "deleted", "field": "status" }] }
}
```

`must` is a hard filter. `should` contributes to score but isn't required. `must_not` excludes documents.

## Geo queries

Field must be indexed as type `geopoint`.

### Geo-distance (radius)

```json
{
  "location": { "lat": 37.7749, "lon": -122.4194 },
  "distance": "50km",
  "field": "location"
}
```

Distance unit options: `m`, `km`, `mi`, `nm`, `ft`.

### Geo-bounding-box

```json
{
  "top_left":     { "lat": 38.0, "lon": -123.0 },
  "bottom_right": { "lat": 37.0, "lon": -121.0 },
  "field": "location"
}
```

### Geo-polygon

```json
{
  "polygon_points": [
    { "lat": 37.9, "lon": -122.5 },
    { "lat": 37.9, "lon": -122.0 },
    { "lat": 37.5, "lon": -122.0 },
    { "lat": 37.5, "lon": -122.5 }
  ],
  "field": "location"
}
```

## Facets

Facets are aggregations on the search result set. Pass them in the `facets` parameter:

```json
{
  "category_facet": { "size": 10, "field": "category" },
  "price_facet": {
    "size": 5,
    "field": "price",
    "numeric_ranges": [
      { "name": "cheap",    "max": 50 },
      { "name": "mid",      "min": 50, "max": 200 },
      { "name": "premium",  "min": 200 }
    ]
  }
}
```

Field must have `docvalues: true` in the index definition.

## Boosting

Add `boost` to any query clause to increase its relevance contribution:

```json
{
  "disjuncts": [
    { "match": "couchbase", "field": "title",       "boost": 3.0 },
    { "match": "couchbase", "field": "description", "boost": 1.0 }
  ]
}
```

## Highlighting

Pass `highlight: { "style": "html", "fields": ["title", "description"] }` to get highlighted snippets in results. Requires `include_term_vectors: true` on the fields.

## Score control

- `explain: true` in the request body to get per-term score breakdown in results (expensive — debug only)
- `score: "none"` to skip scoring entirely and treat FTS as a boolean filter (faster when you don't need ranking)

## Search result shape

```json
{
  "total_hits": 1234,
  "hits": [
    {
      "id": "doc::001",
      "score": 1.87,
      "fields": { "title": "...", "status": "active" },
      "fragments": { "title": ["...highlighted <em>match</em>..."] },
      "locations": { "title": { "couchbase": [{ "pos": 1, "start": 0, "end": 9 }] } }
    }
  ],
  "facets": { "category_facet": { "terms": [...] } },
  "status": { "total": 1, "successful": 1, "failed": 0 }
}
```

`fields` is only populated if you pass the `fields` parameter in the request. `fragments` requires `highlight`. `locations` requires `include_term_vectors`.
