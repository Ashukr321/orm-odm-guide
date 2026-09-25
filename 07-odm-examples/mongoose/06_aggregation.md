## Mongoose Aggregation

This page builds the blog's **analytics and dashboard** endpoints with
aggregation pipelines. Stages and concepts are covered in
[Aggregation](../../06-odm-concepts/08_aggregation.md).

```js
import mongoose from 'mongoose';
import { Post } from './models/post.js';
import { User } from './models/user.js';
import { Tag } from './models/tag.js';

const { Types } = mongoose;
```

> **Remember:** `aggregate()` does **not** cast. Convert ids yourself
> (`new Types.ObjectId(id)`) and pass real `Date` objects.

---

### 1. Dashboard Overview in One Query (`$facet`)

```js
async function dashboard() {
  const [stats] = await Post.aggregate([
    {
      $facet: {
        totals: [
          {
            $group: {
              _id: null,
              posts:     { $sum: 1 },
              published: { $sum: { $cond: ['$published', 1, 0] } },
              views:     { $sum: '$views' },
              comments:  { $sum: { $size: '$comments' } },
            },
          },
          { $project: { _id: 0 } },
        ],
        last7Days: [
          { $match: { createdAt: { $gte: new Date(Date.now() - 7 * 864e5) } } },
          { $count: 'posts' },
        ],
      },
    },
  ]);
  return { ...stats.totals[0], newThisWeek: stats.last7Days[0]?.posts ?? 0 };
}
// { posts: 128, published: 97, views: 45210, comments: 812, newThisWeek: 6 }
```

---

### 2. Top Authors with Names (`$group` + `$lookup`)

```js
async function topAuthors(limit = 5) {
  return Post.aggregate([
    { $match: { published: true } },
    { $group: { _id: '$author', posts: { $sum: 1 }, views: { $sum: '$views' } } },
    { $sort: { views: -1 } },
    { $limit: limit },                                   // limit BEFORE the join: fewer lookups
    { $lookup: { from: 'users', localField: '_id', foreignField: '_id', as: 'author',
                 pipeline: [{ $project: { name: 1 } }] } },
    { $unwind: '$author' },
    { $project: { _id: 0, authorId: '$_id', name: '$author.name', posts: 1, views: 1 } },
  ]);
}
// [{ authorId, name: 'Ashu', posts: 42, views: 20110 }, …]
```

`from` is the **collection** name (`users`), not the model name.

---

### 3. Tag Cloud (`$unwind` Arrays)

```js
async function tagCloud() {
  return Post.aggregate([
    { $match: { published: true } },
    { $unwind: '$tags' },
    { $group: { _id: '$tags', count: { $sum: 1 } } },
    { $sort: { count: -1 } },
    { $limit: 30 },
    { $lookup: { from: 'tags', localField: '_id', foreignField: '_id', as: 'tag' } },
    { $unwind: '$tag' },
    { $project: { _id: 0, name: '$tag.name', count: 1 } },
  ]);
}
```

This can also **recompute** the stored `tag.postCount` values if they
drift:

```js
const counts = await Post.aggregate([
  { $match: { published: true } }, { $unwind: '$tags' }, { $group: { _id: '$tags', n: { $sum: 1 } } },
]);
await Tag.bulkWrite(counts.map((c) => ({ updateOne: { filter: { _id: c._id }, update: { $set: { postCount: c.n } } } })));
```

---

### 4. Posts per Month (Dates)

```js
async function postsPerMonth(year = 2026) {
  return Post.aggregate([
    { $match: { createdAt: { $gte: new Date(`${year}-01-01`), $lt: new Date(`${year + 1}-01-01`) } } },
    {
      $group: {
        _id: { $month: '$createdAt' },
        posts: { $sum: 1 },
        avgViews: { $avg: '$views' },
      },
    },
    { $sort: { _id: 1 } },
    { $project: { _id: 0, month: '$_id', posts: 1, avgViews: { $round: ['$avgViews', 1] } } },
  ]);
}
// [{ month: 1, posts: 14, avgViews: 310.5 }, …]
```

Add `timezone: 'Asia/Kolkata'` to `$month` (as `{ date: '$createdAt',
timezone }`) if months should follow a local calendar instead of UTC.

---

### 5. Most Active Commenters (Embedded Arrays)

```js
async function topCommenters(limit = 10) {
  return Post.aggregate([
    { $unwind: '$comments' },
    { $group: { _id: '$comments.author', comments: { $sum: 1 }, lastAt: { $max: '$comments.createdAt' } } },
    { $sort: { comments: -1 } },
    { $limit: limit },
    { $lookup: { from: 'users', localField: '_id', foreignField: '_id', as: 'user', pipeline: [{ $project: { name: 1 } }] } },
    { $project: { _id: 0, name: { $first: '$user.name' }, comments: 1, lastAt: 1 } },
  ]);
}
```

---

### 6. One Author's Stats (Casting Ids Yourself)

```js
async function authorStats(authorId) {
  const [s] = await Post.aggregate([
    { $match: { author: new Types.ObjectId(authorId) } },          // ← cast!
    {
      $group: {
        _id: null,
        total:     { $sum: 1 },
        published: { $sum: { $cond: ['$published', 1, 0] } },
        views:     { $sum: '$views' },
        bestPost:  { $top: { sortBy: { views: -1 }, output: { title: '$title', views: '$views' } } },
      },
    },
  ]);
  return s ?? { total: 0, published: 0, views: 0, bestPost: null };
}
```

(`$top` needs MongoDB 5.2+.) Validate `authorId` with
`mongoose.isValidObjectId()` first; `new Types.ObjectId('bad')` throws.

---

### 7. Search with Relevance and Pagination

```js
async function search(q, page = 1, size = 10) {
  const [res] = await Post.aggregate([
    { $match: { $text: { $search: q }, published: true } },        // $text must be in the first $match
    { $addFields: { score: { $meta: 'textScore' } } },
    { $sort: { score: -1 } },
    {
      $facet: {
        items: [{ $skip: (page - 1) * size }, { $limit: size }, { $project: { title: 1, slug: 1, score: 1 } }],
        total: [{ $count: 'n' }],
      },
    },
  ]);
  return { items: res.items, total: res.total[0]?.n ?? 0 };
}
```

---

### 8. Per-Author Ranking (`$setWindowFields`)

```js
// Each author's 3 most-viewed posts
await Post.aggregate([
  { $match: { published: true } },
  { $setWindowFields: { partitionBy: '$author', sortBy: { views: -1 }, output: { rank: { $rank: {} } } } },
  { $match: { rank: { $lte: 3 } } },
  { $project: { title: 1, author: 1, views: 1, rank: 1 } },
]);
```

---

### 9. Pre-Computed Daily Stats (`$merge`)

Dashboards that are opened often shouldn't re-scan all posts. Compute
into a summary collection on a schedule (for example, nightly):

```js
await Post.aggregate([
  { $match: { createdAt: { $gte: startOfYesterday, $lt: startOfToday } } },
  { $group: { _id: { $dateToString: { format: '%Y-%m-%d', date: '$createdAt' } },
              posts: { $sum: 1 }, views: { $sum: '$views' } } },
  { $merge: { into: 'daily_stats', on: '_id', whenMatched: 'replace', whenNotMatched: 'insert' } },
]);

// The dashboard reads the small collection
const stats = await mongoose.connection.db.collection('daily_stats').find().sort({ _id: -1 }).limit(30).toArray();
```

---

### 10. Exposing It: a Stats Router

```js
import { Router } from 'express';
import { isValidObjectId } from 'mongoose';

export const stats = Router();

stats.get('/overview', async (_req, res) => res.json(await dashboard()));
stats.get('/top-authors', async (req, res) => res.json(await topAuthors(Math.min(Number(req.query.limit) || 5, 50))));
stats.get('/tags', async (_req, res) => res.json(await tagCloud()));
stats.get('/authors/:id', async (req, res) => {
  if (!isValidObjectId(req.params.id)) return res.sendStatus(404);
  res.json(await authorStats(req.params.id));
});
```

Cap client-supplied limits, and cache expensive endpoints (for example,
for 60 seconds) if they're public.

---

### Performance Checklist

| Do | Why |
|---|---|
| `$match` first (and `$sort` early) | Only the start of the pipeline can use indexes |
| `$limit` before `$lookup` | Join 5 documents, not 50,000 |
| `$project` / `pipeline` inside `$lookup` | Move less data |
| Check with `.explain('executionStats')` | See whether an index is used |
| `allowDiskUse(true)` for big sorts/groups on older servers | 100 MB per-stage memory limit |
| `$merge` into summary collections | Don't recompute heavy stats per request |
| Aggregate middleware for soft delete | `pre('find')` hooks don't apply to `aggregate()` |

```js
const plan = await Post.aggregate([{ $match: { published: true } }, { $sort: { createdAt: -1 } }, { $limit: 10 }])
  .explain('executionStats');
```

---

### Interview-Ready Summary

- **`$facet`** returns several results (totals, recent counts, page +
  total) in one query.
- **`$group` + `$sort` + `$limit` + `$lookup`** builds leaderboards; limit
  **before** joining, and `from` is the collection name.
- **`$unwind`** turns arrays (tags, embedded comments) into rows for
  grouping.
- `aggregate()` **doesn't cast**: use `new Types.ObjectId(id)` and real
  `Date`s, and validate ids first.
- `$text` + `textScore` for ranked search, `$setWindowFields` for
  per-group ranks, and **`$merge`** to pre-compute dashboards.
- Performance: `$match` first, `$project` early, `explain()`, and cap
  client limits.

Compare: [Prisma Advanced Queries](../../04-orm-examples/prisma/05_advanced-queries.md)
