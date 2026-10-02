# Route Matching and Tree

<!-- Maintained by Tidewiki. Edits are kept on later updates; wrap text in tidewiki:keep markers to freeze it. -->

Chi uses a **Patricia Radix trie** to organize and match routes with high efficiency. When you register a route, it is stored in a tree structure optimized for fast lookup during request handling.

## Overview

The router maintains an internal tree where each node represents a segment of a route path. Routes are matched by traversing this tree based on the incoming request path, extracting parameters along the way, and finding the appropriate handler. [`tree.go:3-5`](../../tree.go#L3-L5)

## Tree Structure

The tree is organized into nodes of different types: [`tree.go:90-95`](../../tree.go#L90-L95)

- **Static** (`ntStatic`): literal path segments like `/users` or `/articles`
- **Param** (`ntParam`): named parameters like `{id}` that match anything up to the next delimiter
- **Regexp** (`ntRegexp`): parameters with regex constraints like `{id:[0-9]+}`
- **CatchAll** (`ntCatchAll`): wildcard segments using `*` that match everything remaining

Each node stores prefix strings to enable radix tree compression. [`tree.go:97-122`](../../tree.go#L97-L122)

## Route Insertion

When you register a route like `r.Get("/users/{id}/posts", handler)`, chi parses the pattern, splits it into segments, and inserts them into the tree. [`tree.go:148-238`](../../tree.go#L148-L238)

Routes are inserted using the `InsertRoute` method, which:
1. Parses the pattern into static and dynamic segments
2. Traverses the tree to find or create nodes for each segment
3. Compresses common prefixes to minimize tree depth
4. Stores the handler at the appropriate leaf node

Example usage:

```go
r := chi.NewRouter()
r.Get("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
  id := chi.URLParam(r, "id")
  w.Write([]byte("User: " + id))
})
```

## Route Matching

When a request arrives, chi calls `FindRoute` to traverse the tree and locate the matching handler. [`tree.go:383-406`](../../tree.go#L383-L406)

The matching process:
1. Starts at the root node and works through the request path
2. Tries to match against static nodes first (exact prefix match)
3. Then attempts param and regexp nodes, extracting values as it goes
4. Falls back to catch-all nodes if no exact match is found
5. Returns the handler, parameter keys, and parameter values

The matching is **recursive** and **multi-dimensional** because it must consider different node types at each level. [`tree.go:410-557`](../../tree.go#L410-L557)

Example matching flow for `/users/123/posts`:
1. Match static `/users/` prefix
2. Match param `{id}` node, extract value `123`
3. Match static `/posts` prefix
4. Return the handler registered for that pattern

```go
// The router finds the handler and extracts parameters automatically
// When your handler is called, URLParam retrieves the matched values
r.Get("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
  id := chi.URLParam(r, "id")  // id = "123"
})
```

## Parameter Extraction

As the tree is traversed during matching, parameter values are collected in the route context. Each node knows which parameters it represents via `paramKeys`, allowing chi to build a complete key-value mapping. [`tree.go:128-137`](../../tree.go#L128-L137)

For routes with regex constraints like `/articles/{id:[0-9]+}`, chi compiles the regex and validates segments during matching before accepting them as parameter values. [`tree.go:457-460`](../../tree.go#L457-L460)

## Node Ordering

Child nodes are kept sorted by their label (first byte) to enable binary search during lookups. [`tree.go:834-837`](../../tree.go#L834-L837) For param nodes, those with `/` as the tail delimiter are pushed to the end so they're tried last, ensuring more specific routes are matched first. [`tree.go:841-848`](../../tree.go#L841-L848)

## Pattern Parsing

Route patterns are broken down into segments using `patNextSegment`, which identifies the segment type and boundaries. [`tree.go:735-801`](../../tree.go#L735-L801) This allows chi to handle complex patterns like `/articles/{id:[a-z0-9-]+}/posts/{slug}` by parsing each component independently.

## Preventing Duplicate Methods in Allow Header

When a route is not found but other methods are registered on the same path, chi collects those allowed methods for the 405 response. [`tree.go:482-484`](../../tree.go#L482-L484) and [`tree.go:530-532`](../../tree.go#L530-L532) To prevent duplicate method entries in the Allow header, chi uses `slices.Contains` to check if a method type is already in the list before appending it.

## Routes and Mount Stubs

When using `Mount()` or `Route()`, chi installs a synthetic "stub" handler on the mount pattern to connect it to the subrouter. The `routes()` method uses the `equalHandlers` function [`tree.go:696-714`](../../tree.go#L696-L714) to identify and hide only the stub handler itself, while preserving any real handler that shares the same pattern. This prevents internal routing infrastructure from leaking into the public Routes API.

The tree structure and matching algorithm together ensure that:
- Route lookup is **O(k)** where k is the depth of the pattern (very fast)
- Memory usage is minimized through prefix compression
- Specific routes are preferred over general ones
- Both exact matches and parameterized routes work correctly

[Routing Basics](routing-basics.md) explains the high-level API for defining routes. [Getting Started](getting-started.md) shows how to build a complete router.
