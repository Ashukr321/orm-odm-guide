## Aggregation

### What Is the Aggregation Pipeline

The **aggregation pipeline** is MongoDB's tool for analytics and data
transformation: grouping, counting, joining, reshaping. Documents flow
through a list of **stages**; each stage transforms the stream and passes
it on.

```
posts ──► $match ──► $group ──► $sort ──► $limit ──► results
          filter     combine    order     top N
```

```js
const topAuthors = await Post.aggregate([
  { $match: { published: true } },
  { $group: { _id: '$author', posts: { $sum: 1 }, views: { $sum: '$views' } } },
  { $sort: { views: -1 } },
  { $limit: 5 },
]);
// [{ _id: ObjectId('66a…'), posts: 12, views: 9120 }, …]
```

| SQL | Aggregation stage |
|---|---|
| `WHERE` | `$match` |
| `GROUP BY` + `COUNT/SUM/AVG` | `$group` + `$sum`, `$avg`, `$min`, `$max` |
| `SELECT col, expr AS x` | `$project`, `$addFields` / `$set` |
| `ORDER BY` / `LIMIT` / `OFFSET` | `$sort`, `$limit`, `$skip` |
| `JOIN` | `$lookup` |
| `HAVING` | `$match` after `$group` |
| Window functions | `$setWindowFields` |

---

### Key Stages

| Stage | Does |
|---|---|
| `$match` | Filter documents (uses indexes when first) |
| `$project` | Include/exclude/compute fields |
| `$addFields` / `$set` | Add computed fields, keep the rest |
| `$group` | Group by a key and accumulate |
| `$sort`, `$limit`, `$skip` | Order and paginate |
| `$unwind` | One document per array element |
| `$lookup` | Join another collection |
| `$facet` | Several sub-pipelines on the same input |
| `$bucket` / `$bucketAuto` | Group into ranges |
| `$count` | Count documents |
| `$setWindowFields` | Running totals, ranks, moving averages |
| `$out` / `$merge` | Write results to a collection |

---

### Mongoose Specifics (Read This First)

`Model.aggregate()` sends the pipeline **as is**:

- **No casting.** Strings are **not** converted to `ObjectId` or `Date`.
- **No schema defaults, getters or virtuals.** Results are plain objects.
- Query middleware (`pre('find')`) doesn't run; `pre('aggregate')` does.

```js
// ❌ Matches nothing: the author field holds ObjectIds, not strings
await Post.aggregate([{ $match: { author: req.params.userId } }]);

// ✅ Cast yourself
const { Types } = require('mongoose');
await Post.aggregate([{ $match: { author: new Types.ObjectId(req.params.userId) } }]);
```

---

### Example: Tag Counts (`$unwind` + `$group`)

```js
const tagCounts = await Post.aggregate([
  { $match: { published: true } },
  { $unwind: '$tags' },                                  // one doc per tag
  { $group: { _id: '$tags', count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $project: { _id: 0, tag: '$_id', count: 1 } },
]);
// [{ tag: 'mongodb', count: 42 }, { tag: 'node', count: 30 }, …]
```

---

### Example: Posts per Month

```js
const perMonth = await Post.aggregate([
  { $match: { createdAt: { $gte: new Date('2026-01-01') } } },
  {
    $group: {
      _id: { $dateToString: { format: '%Y-%m', date: '$createdAt' } },
      posts: { $sum: 1 },
      avgComments: { $avg: { $size: '$comments' } },
    },
  },
  { $sort: { _id: 1 } },
]);
// [{ _id: '2026-01', posts: 18, avgComments: 3.2 }, …]
```

---

### Example: Joining with `$lookup`

```js
const posts = await Post.aggregate([
  { $match: { published: true } },
  {
    $lookup: {
      from: 'users',               // collection name, not model name
      localField: 'author',
      foreignField: '_id',
      as: 'author',
    },
  },
  { $unwind: '$author' },          // array → single object
  { $match: { 'author.role': 'admin' } },   // filter by the joined data
  { $project: { title: 1, 'author.name': 1, 'author.email': 1 } },
]);
```

**`$lookup` with a pipeline** (filter and shape the joined documents):

```js
{
  $lookup: {
    from: 'posts',
    let: { userId: '$_id' },
    pipeline: [
      { $match: { $expr: { $eq: ['$author', '$$userId'] }, published: true } },
      { $sort: { createdAt: -1 } },
      { $limit: 3 },
      { $project: { title: 1 } },
    ],
    as: 'latestPosts',
  },
}
```

---

### Example: Pagination + Total in One Query (`$facet`)

```js
const [result] = await Post.aggregate([
  { $match: { published: true } },
  { $sort: { createdAt: -1 } },
  {
    $facet: {
      items: [{ $skip: 20 }, { $limit: 10 }, { $project: { title: 1, createdAt: 1 } }],
      total: [{ $count: 'count' }],
    },
  },
]);
const items = result.items;
const total = result.total[0]?.count ?? 0;
```

---

### Example: Ranking with `$setWindowFields`

```js
await Post.aggregate([
  { $match: { published: true } },
  {
    $setWindowFields: {
      partitionBy: '$author',
      sortBy: { views: -1 },
      output: { rankInAuthor: { $rank: {} } },
    },
  },
  { $match: { rankInAuthor: { $lte: 3 } } },   // each author's top 3 posts
]);
```

---

### The Mongoose Aggregate Builder

```js
const res = await Post.aggregate()
  .match({ published: true })
  .group({ _id: '$author', posts: { $sum: 1 } })
  .sort({ posts: -1 })
  .limit(5);
```

Same pipeline, chainable syntax.

---

### Performance Tips

1. **`$match` (and `$sort`) first**: only early stages can use indexes.
2. **`$project` early** to drop fields you don't need.
3. **`$limit` right after `$sort`** lets MongoDB keep only the top N in memory.
4. Stages have a **100 MB memory limit**; older servers need
   `allowDiskUse: true` for big sorts and groups:

```js
await Post.aggregate(pipeline).allowDiskUse(true);
```

5. Check the plan:

```js
await Post.aggregate(pipeline).explain('executionStats');
```

6. For dashboards hit often, pre-compute with `$merge` into a summary
   collection on a schedule.

---

### Interview-Ready Summary

- The **aggregation pipeline** runs documents through stages: `$match`,
  `$group`, `$project`, `$sort`, `$limit`, `$unwind`, `$lookup`, `$facet`,
  `$setWindowFields`.
- It's MongoDB's equivalent of SQL `WHERE`, `GROUP BY`, `JOIN`, `HAVING` and
  window functions.
- In Mongoose, **`aggregate()` doesn't cast**: convert ids with
  `new Types.ObjectId()`. Results are plain objects.
- `$lookup` is a server-side join (use `from` = collection name); `$facet`
  gives page + total in one query.
- Put `$match`/`$sort` first for indexes, `$limit` after `$sort`, watch the
  100 MB stage limit, and use `explain()`.
