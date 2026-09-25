## Mongoose Relationships

### Two Tools: Embed or Reference

In MongoDB, a relationship is modelled by **embedding** the related data
inside a document or **referencing** it by `ObjectId`. The choice depends on
how the data is read and how big it can grow.

| Relationship | Blog example | Chosen design |
|---|---|---|
| 1:1 | User ↔ settings | **Embed** a sub-object |
| 1:few | Post → comments (capped) | **Embed** an array of subdocuments |
| 1:many | User → posts | **Reference** from the child (`post.author`) |
| M:N | Posts ↔ tags | **Array of references** on one side (`post.tags`) |
| Self | User ↔ followers | Separate `follows` collection |

Concepts: [Collections](../../06-odm-concepts/04_collections.md) ·
[What Is ODM](../../05-odm/01_what-is-odm.md)

---

### One-to-One: Embed

Settings are always read with the user and never on their own, so they
live inside the user document.

```js
const userSchema = new Schema({
  email: String,
  settings: {
    theme:         { type: String, enum: ['light', 'dark'], default: 'light' },
    emailDigest:   { type: Boolean, default: true },
    language:      { type: String, default: 'en' },
  },
});

await User.updateOne({ _id: id }, { $set: { 'settings.theme': 'dark' } });
const user = await User.findById(id).select('email settings');
```

Use a **referenced** 1:1 (a separate collection with a unique `user` field)
only when the other side is large or sensitive (for example, a big
profile or billing details read on few pages).

---

### One-to-Few: Embedded Array

Comments are read with the post and capped at 500 per post (see
[Schema](02_schema.md)).

```js
// Add
await Post.updateOne(
  { _id: postId },
  { $push: { comments: { body: 'Great read!', author: userId } } },
  { runValidators: true }
);

// Edit one comment (positional $ operator)
await Post.updateOne(
  { _id: postId, 'comments._id': commentId, 'comments.author': userId },   // only the author
  { $set: { 'comments.$.body': 'Edited' } }
);

// Remove
await Post.updateOne({ _id: postId }, { $pull: { comments: { _id: commentId } } });

// Latest 5 comments without loading them all
await Post.findById(postId).select({ title: 1, comments: { $slice: -5 } });

// Posts a user commented on
await Post.find({ 'comments.author': userId }).select('title');
```

Or work with subdocuments on a loaded post:

```js
const post = await Post.findById(postId);
post.comments.id(commentId).body = 'Edited';
post.comments.push({ body: 'Another', author: userId });
await post.save();
```

---

### One-to-Many: Reference from the Child

A user can write thousands of posts, so posts are their own collection and
each post stores its `author`.

```js
// Create
await Post.create({ title: 'Hello', author: user._id });

// All posts by a user (indexed: { author: 1, createdAt: -1 })
await Post.find({ author: user._id }).sort({ createdAt: -1 });

// The author of a post
await Post.findById(postId).populate('author', 'name email');

// A user's posts from the user side, via the virtual defined in the schema
await User.findById(userId).populate({ path: 'posts', select: 'title', options: { sort: { createdAt: -1 } } });
```

**Don't** also keep `user.posts: [ObjectId]`. That array grows without
limit and must be kept in sync by hand. The virtual populate gives you the
same API with a single source of truth.

---

### Many-to-Many: Array of References

A post has a few tags; a tag has many posts. Store the references on the
side with the **small, bounded** list: `post.tags`.

```js
// Tag a post (no duplicates)
await Post.updateOne({ _id: postId }, { $addToSet: { tags: { $each: [mongoId, nodeId] } } });

// Untag
await Post.updateOne({ _id: postId }, { $pull: { tags: nodeId } });

// Posts with a tag (multikey index on tags)
await Post.find({ tags: mongoId, published: true });

// Posts with ALL of these tags / ANY of them
await Post.find({ tags: { $all: [mongoId, nodeId] } });
await Post.find({ tags: { $in: [mongoId, nodeId] } });

// A post with its tag names
await Post.findById(postId).populate('tags', 'name');
```

Alternative: store **tag names** instead of ids (`tags: ['mongodb', 'node']`).
It's simpler and needs no populate, but renaming a tag means updating
every post. That's a fine trade-off when renames are rare.

---

### Self-Referencing Many-to-Many: Followers

Follower lists can be huge (think celebrities), so each "follow" is its own
small document.

```js
const followSchema = new Schema(
  {
    follower:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
    following: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  },
  { timestamps: true }
);
followSchema.index({ follower: 1, following: 1 }, { unique: true });   // no double follows
followSchema.index({ following: 1, createdAt: -1 });                  // "who follows X"
export const Follow = model('Follow', followSchema);

await Follow.create({ follower: me, following: them });
const followers = await Follow.find({ following: them }).populate('follower', 'name').limit(50);
const followerCount = await Follow.countDocuments({ following: them });
```

This is the MongoDB version of a SQL join table.

---

### Extended Reference: Copy What You Show

The feed shows each post's author name. Instead of populating on every
request, copy the name into the post:

```js
author: {
  _id:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
  name: String,
},

// When a user renames (rare), update their copies
await Post.updateMany({ 'author._id': userId }, { $set: { 'author.name': newName } });
```

Faster reads, at the cost of duplicated data you must keep in sync.

---

### Keeping Both Sides Consistent: Transactions

Publishing a post also updates each tag's `postCount`. Both writes must
succeed together, so use a **transaction** (this needs a replica set; see
[Setup](01_setup.md)).

```js
import mongoose from 'mongoose';

async function publishPost(postId) {
  await mongoose.connection.transaction(async (session) => {
    const post = await Post.findById(postId).session(session).orFail();
    if (post.published) return;

    post.published = true;
    await post.save({ session });

    await Tag.updateMany({ _id: { $in: post.tags } }, { $inc: { postCount: 1 } }, { session });
  });
  // committed; if anything threw, both writes were rolled back (and transient errors retried)
}
```

- Pass **`session`** to **every** operation inside, or it runs outside the
  transaction.
- Keep transactions short, and prefer single-document atomic updates when
  they're enough. A single-document update is always atomic.

---

### Deleting Related Data (Cascades)

MongoDB has no foreign keys, so nothing cascades automatically.

```js
async function deleteUser(userId) {
  await mongoose.connection.transaction(async (session) => {
    const posts = await Post.find({ author: userId, published: true }).select('tags').session(session);
    const tagOps = posts.flatMap((p) => p.tags).map((tagId) => ({
      updateOne: { filter: { _id: tagId }, update: { $inc: { postCount: -1 } } },
    }));
    if (tagOps.length) await Tag.bulkWrite(tagOps, { session });
    await Post.deleteMany({ author: userId }, { session });
    await Post.updateMany({}, { $pull: { comments: { author: userId } } }, { session });
    await Follow.deleteMany({ $or: [{ follower: userId }, { following: userId }] }, { session });
    await User.deleteOne({ _id: userId }, { session });
  });
}
```

Or soft-delete the user and clean up in a background job.

---

### Interview-Ready Summary

- **Embed** 1:1 and 1:few data that's read with the parent (settings,
  capped comments); update it with `$set`, `$push`, positional `$` and
  `$pull`.
- **Reference from the child** for 1:many (`post.author`), and read the
  reverse side with **virtual populate**, not a growing id array.
- **M:N**: an array of references on the bounded side (`post.tags`) with
  `$addToSet`/`$pull`/`$all`/`$in`, or a separate collection (`follows`)
  when both sides are large.
- **Extended references** copy display fields to avoid lookups.
- No foreign keys: keep multi-document changes and cascades consistent with
  **transactions** (`connection.transaction`, pass `session` everywhere).

Next: [Population](05_population.md)
