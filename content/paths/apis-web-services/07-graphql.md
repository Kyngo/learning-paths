---
title: "GraphQL"
weight: 7
---

# GraphQL

GraphQL is a query language for APIs and a runtime for executing those queries. Unlike REST, it lets clients request exactly the data they need in a single request, eliminating over-fetching and under-fetching.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| Schema | Defines the types, queries, mutations, and subscriptions available |
| Query | Read operation — fetches data |
| Mutation | Write operation — creates, updates, or deletes data |
| Subscription | Real-time operation — pushes data to clients via WebSocket |
| Resolver | Function that fetches the data for a specific field |
| Type | Defines the shape of data (object, scalar, enum, interface, union) |

---

## Schema Definition Language (SDL)

### Scalar Types

GraphQL has five built-in scalars: `Int`, `Float`, `String`, `Boolean`, `ID`.

### Object Types

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  body: String!
  author: User!
  comments: [Comment!]!
  publishedAt: DateTime
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
}
```

The `!` suffix means non-nullable. `[Post!]!` means a non-null list of non-null posts.

### Enums

```graphql
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

### Interfaces and Unions

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

union SearchResult = User | Post | Comment
```

---

## Queries

```graphql
type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
  search(term: String!): [SearchResult!]!
}
```

### Client Query Example

```graphql
query GetUserWithPosts($userId: ID!) {
  user(id: $userId) {
    name
    email
    posts {
      title
      publishedAt
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
```

The client specifies exactly which fields to return — no over-fetching.

---

## Mutations

```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  likePost(id: ID!): Post!
}

input CreatePostInput {
  title: String!
  body: String!
  status: PostStatus = DRAFT
}

input UpdatePostInput {
  title: String
  body: String
  status: PostStatus
}
```

### Mutation Usage

```graphql
mutation CreateNewPost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    status
  }
}
```

Variables:
```json
{
  "input": {
    "title": "GraphQL Fundamentals",
    "body": "GraphQL is a query language...",
    "status": "PUBLISHED"
  }
}
```

---

## Subscriptions

Subscriptions provide real-time updates via WebSocket (typically using the `graphql-ws` protocol).

```graphql
type Subscription {
  postCreated: Post!
  commentAdded(postId: ID!): Comment!
  userStatusChanged(userId: ID!): User!
}
```

### Client Subscription

```graphql
subscription OnNewComment($postId: ID!) {
  commentAdded(postId: $postId) {
    id
    text
    author {
      name
    }
  }
}
```

---

## Resolvers

Resolvers are functions that populate each field in the schema. They receive four arguments:

```javascript
// JavaScript/Node.js resolver example
const resolvers = {
  Query: {
    user: (parent, args, context, info) => {
      return context.db.users.findById(args.id);
    },
    users: (parent, { limit, offset }, context) => {
      return context.db.users.findAll({ limit, offset });
    },
  },
  User: {
    posts: (user, args, context) => {
      return context.db.posts.findByAuthor(user.id);
    },
  },
  Post: {
    author: (post, args, context) => {
      return context.db.users.findById(post.authorId);
    },
  },
};
```

| Argument | Description |
|----------|-------------|
| `parent` | Result of the parent resolver (for nested fields) |
| `args` | Arguments passed to the field |
| `context` | Shared object (auth, database connections, DataLoaders) |
| `info` | Query AST and schema metadata |

---

## The N+1 Problem and DataLoader

### The Problem

When resolving a list, each item triggers its own database query:

```
Query: users → 1 query (fetches 50 users)
  User.posts → 50 queries (one per user)
```

This is the **N+1 problem**: 1 query for the list + N queries for each related field.

### DataLoader Solution

DataLoader batches and caches individual loads within a single request:

```javascript
const DataLoader = require('dataloader');

// Batch function: receives array of keys, returns array of results
const postsByAuthorLoader = new DataLoader(async (authorIds) => {
  const posts = await db.posts.findByAuthors(authorIds);
  // Group by authorId, maintaining key order
  return authorIds.map(id => posts.filter(p => p.authorId === id));
});

const resolvers = {
  User: {
    posts: (user, args, context) => {
      return context.loaders.postsByAuthor.load(user.id);
    },
  },
};
```

**Result:** Instead of 50 individual queries, DataLoader batches them into 1 query with `WHERE author_id IN (...)`.

### Python Equivalent (Strawberry + aiodataloader)

```python
from aiodataloader import DataLoader

class PostsByAuthorLoader(DataLoader):
    async def batch_load_fn(self, author_ids: list[str]) -> list[list]:
        posts = await db.posts.find_by_authors(author_ids)
        grouped = {aid: [] for aid in author_ids}
        for post in posts:
            grouped[post.author_id].append(post)
        return [grouped[aid] for aid in author_ids]
```

---

## Fragments

Fragments allow reusing field selections across queries:

```graphql
fragment UserBasicInfo on User {
  id
  name
  email
}

fragment PostSummary on Post {
  id
  title
  publishedAt
}

query Dashboard {
  me {
    ...UserBasicInfo
    posts {
      ...PostSummary
    }
  }
  latestPosts {
    ...PostSummary
    author {
      ...UserBasicInfo
    }
  }
}
```

### Inline Fragments (for Unions/Interfaces)

```graphql
query Search($term: String!) {
  search(term: $term) {
    ... on User {
      name
      email
    }
    ... on Post {
      title
      body
    }
    ... on Comment {
      text
    }
  }
}
```

---

## Pagination — Relay Specification

The Relay cursor-based pagination spec provides a consistent pagination interface:

```graphql
type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### Usage

```graphql
query PaginatedPosts($cursor: String) {
  posts(first: 10, after: $cursor) {
    edges {
      node {
        title
        publishedAt
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

| Parameter | Description |
|-----------|-------------|
| `first` | Number of items from the start |
| `after` | Cursor — fetch items after this point |
| `last` | Number of items from the end |
| `before` | Cursor — fetch items before this point |

---

## GraphQL vs REST — When to Use Which

| Factor | GraphQL | REST |
|--------|---------|------|
| Data fetching | Precise — client specifies fields | Fixed responses per endpoint |
| Over-fetching | Eliminated | Common |
| Under-fetching | Eliminated (one request) | Requires multiple endpoints |
| Caching | Complex (POST-based, per-query) | Simple (HTTP caching, ETags) |
| File uploads | Requires multipart spec or separate endpoint | Native support |
| Real-time | Built-in subscriptions | Requires separate WebSocket setup |
| Versioning | Schema evolution (deprecation) | URL or header versioning |
| Tooling | Schema-driven (introspection, codegen) | OpenAPI/Swagger |
| Learning curve | Higher | Lower |
| Best for | Complex UIs, mobile apps, BFFs | CRUD APIs, public APIs, microservices |

### Choose GraphQL When

- Clients need **flexible queries** (dashboards, mobile vs web)
- You have **deeply nested or related data**
- You want a **single endpoint** serving multiple client types
- Rapid frontend iteration is a priority

### Choose REST When

- You need **simple CRUD** with predictable responses
- **HTTP caching** is critical for performance
- You're building a **public API** (broader familiarity)
- **File upload/download** is a primary use case

---

## Error Handling

GraphQL returns errors alongside data (partial responses are valid):

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "posts": null
    }
  },
  "errors": [
    {
      "message": "Not authorized to view posts",
      "path": ["user", "posts"],
      "extensions": {
        "code": "FORBIDDEN"
      }
    }
  ]
}
```

---

## Security Considerations

| Threat | Mitigation |
|--------|-----------|
| Deeply nested queries (DoS) | Query depth limiting |
| Expensive queries | Query complexity analysis |
| Introspection in production | Disable introspection |
| Batched queries | Limit batch size |
| Unauthorized field access | Field-level authorization in resolvers |

---

## Key Takeaways

- GraphQL's **schema is the contract** — it serves as documentation, validation, and type system in one
- The **N+1 problem** is the most common performance pitfall; DataLoader is the standard solution
- **Relay-style pagination** (cursor-based) is more robust than offset/limit for large datasets
- GraphQL excels when **clients have diverse data needs** — mobile and web consuming the same API differently
- **REST remains better** for simple CRUD, public APIs, and when HTTP caching is critical
- Always implement **depth limiting and complexity analysis** to prevent abusive queries
- Fragments and variables make queries **reusable and maintainable** on the client side
