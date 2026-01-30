# GerryChain Graph Migration Guide

This is a quick “what changed / what to do instead” guide for users of GerryChain's `Graph` class
who need to migrate code from versions prior to 1.0.0 that relied on `Graph` being a subclass of 
`networkx.Graph`.

---

## “Old → New” quick replacements

| Old pattern                        | New pattern                                                   |
| ---------------------------------- | ------------------------------------------------------------- |
| `g.nodes[n][k]`                    | `g.node_data(n)[k]`                                           |
| `g.edges[u, v][k]`                 | `g.edge_data((u,v))[k]`                                       |
| `nx.alg(g)`                        | `nx.alg(g.get_nx_graph())` or `nx.alg(g.to_networkx_graph())` |
| `sub = g.subgraph(S)` (assume IDs) | `sub = g.subgraph(S)` then translate results if RX-backed     |


## New features

There are a lot of new convenience methods on `Graph` that work regardless of backend:

| New Method                                                      | Description                                                                                                |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `Graph.from_networkx(nx_graph)`                                 | Wrap a user-created `networkx.Graph` as a GerryChain `Graph` (NX-backed).                                  |
| `Graph.from_json(path)`                                         | Load a graph from NetworkX adjacency JSON (NX-backed wrapper).                                             |
| `Graph.nodes`                                                   | List of node IDs (legacy-friendly; good for iteration/printing).                                           |
| `Graph.edges`                                                   | Set of edge endpoint tuples `(u, v)` (portable across backends).                                           |
| `Graph.node_indices`                                            | Set of node IDs (NX labels or RX integer ids).                                                             |
| `Graph.edge_indices`                                            | Set of edge IDs (NX: `(u,v)`; RX: integer edge indices).                                                   |
| `Graph.neighbors(node_id)`                                      | Neighbor node IDs as a list (portable across backends).                                                    |
| `Graph.degree(node_id)`                                         | Degree of a node (portable across backends).                                                               |
| `Graph.node_data(node_id)`                                      | Node attribute dict (portable; replacement for `graph.nodes[n][...]`).                                     |
| `Graph.edge_data(edge_id)`                                      | Edge attribute dict (portable; replacement for `graph.edges[u,v][...]`).                                   |
| `Graph.get_edge_id_from_edge((u, v))`                           | Convert an edge tuple into an “edge id” (NX: returns tuple; RX: returns integer edge index).               |
| `Graph.get_edge_from_edge_id(edge_id)`                          | Convert an “edge id” back into an edge tuple `(u, v)` (NX no-op; RX looks up endpoints).                   |
| `Graph.subgraph(nodes)`                                         | Create a subgraph wrapper. **Note:** RX subgraphs renumber nodes, so translate results back.               |
| `Graph.num_connected_components()`                              | Count connected components.                                                                                |
| `Graph.laplacian_matrix()`                                      | Return the (sparse) Laplacian matrix (portable across backends).                                           |
| `Graph.normalized_laplacian_matrix()`                           | Return the (sparse) normalized Laplacian matrix (portable across backends).                                |
| `Graph.minimum_spanning_tree_from_edge_weight(attr)`            | Compute a minimum spanning tree using edge weight attribute `attr` (portable across backends).             |
| `Graph.islands`                                                 | Set of degree-0 nodes (often used for validation/warnings).                                                |

For a full list of new methods, see [New convenience functions](#new-convenience-functions) at
the end of this document.

---

## 1) Big conceptual change

### Old

* `Graph` **was** a `networkx.Graph` subclass.
* Anything that worked on a `networkx.Graph` usually worked on `Graph`.

### New

* `Graph` is a **wrapper** around either:

  * an embedded **NetworkX** graph (`_nx_graph`), or
  * an embedded **RustworkX** graph (`_rx_graph`).

* When working with a `Graph`, object constructed using a NetworkX graph are **NX-backed**, so 
  many of the attribute functions that are defined on `networkx.Graph` will work as expected.
  However, when using a function defined from the `NetworkX` library, you must first extract the
  underlying `networkx.Graph` object using `G.get_nx_graph()`.

* The graphs attached to `Partition` and `MarkovChain` objects are always **RX-backed** 
  (converted automatically), so calling NetworkX functions on them inside of something like an 
  updater will **not work**. Writing custom updaters using other updaters should still work fine.

---

## 2) Graph construction

### Old

```python
g = Graph(nx_graph)
g = Graph.from_networkx(nx_graph)
g = Graph.from_json("g.json")
g = Graph.from_file("shapefile.shp")
```

### New (same ideas, slightly different behavior)

```python
g = Graph.from_networkx(nx_graph)      # wraps NX
g = Graph.from_null_networkx()         # empty NX graph
g = Graph.from_json("g.json")          # loads to NX-backed Graph
g = Graph.from_file("file.geojson")    # NX-backed Graph
g = Graph.from_geodataframe(gdf)       # NX-backed Graph
```

---

## 3) “Is it NX or RX?” and getting the underlying graph

### New

```python
g.is_nx_graph()
g.is_rx_graph()

nxg = g.get_nx_graph()         # only if NX-backed
rxg = g.get_rx_graph()         # only if RX-backed

nxg2 = g.to_networkx_graph()   # works for RX-backed too (rebuilds NX graph)
```

**Migration tip:** if you have existing code that expects a true `networkx.Graph`, call 
`g.get_nx_graph()` (pre-conversion) or you can call `g.to_networkx_graph()` (post-conversion)
but it should be unnecessary to use the latter in almost every case.

---

## 4) Nodes and edges: views vs IDs

### Old

* `graph.nodes` → NetworkX NodeView
* `graph.nodes()` → NetworkX NodeView
* `graph.edges` → EdgeView
* `graph.edges()` → EdgeView
* Iteration: `for n in graph:` yielded nodes.

### New

* `graph.nodes` → **list of node IDs**
* `graph.edges` → **set of `(u, v)` tuples**
* Iteration still yields node IDs.

DEPRECATED:

* `graph.nodes()`
* `graph.edges()`


Also added explicit “ID sets”:

```python
g.node_indices   # set of node IDs (NX nodes or RX integer ids)
g.edge_indices   # set of edge IDs (NX: edge tuples, RX: integer edge indices)
```

---

## 5) Node attributes access (most common break)

### Old

```python
pop = g.nodes[n]["population"]
g.nodes[n]["foo"] = 1
```

### New (works in both NX and RX)

```python
pop = g.node_data(n)["population"]
g.node_data(n)["foo"] = 1
```

You can still access the underlying NX graph directly and use old patterns, if needed:

```python
g.get_nx_graph().nodes[n]["population"]
```

---

## 6) Edge attributes access + edge IDs (second most common break, but unlikely to be noticed)

In NX, an “edge id” is basically `(u, v)`.
In RX, an “edge id” is an **integer index**.

### Use these helpers to write backend-agnostic code

```python
eid = g.get_edge_id_from_edge((u, v))      # returns (u,v) in NX, int in RX
edge = g.get_edge_from_edge_id(eid)        # returns (u,v) in both cases

w = g.edge_data(eid)["weight"]
g.edge_data(eid)["weight"] = w + 1
```

### Old

```python
w = g.edges[u, v]["weight"]
```

---

## 7) Neighbors / degree (now explicit methods)

### Old

```python
list(g.neighbors(n))   # networkx.Graph method
g.degree[n]            # or g.degree(n)
```

### New

```python
nbrs = g.neighbors(n)  # returns list
deg = g.degree(n)      # returns int
```

These work for both NX and RX-backed graphs.

---

## 8) Subgraphs: major semantic change in RX

### Old (NetworkX)

* `sub = g.subgraph(nodes)` preserved node IDs.

### New

* NX-backed subgraphs: IDs preserved (same as before)
* RX-backed subgraphs: node IDs are **renumbered** to `0..k-1`

To deal with this safely, the wrapper maintains maps:

* `_node_id_to_parent_node_id_map` (subgraph id → parent graph id)
* `_node_id_to_original_nx_node_id_map` (internal id → original NX id)

### If your algorithm runs on a subgraph and returns node-keyed results:

**Translate flips:**

```python
translated = sub.translate_subgraph_node_ids_for_flips(flips)
```

**Translate sets:**

```python
translated_set = sub.translate_subgraph_node_ids_for_set_of_nodes(node_set)
```

**Get original labels (for reporting / debugging):**

```python
orig = g.original_nx_node_id_for_internal_node_id(internal_id)
```

---

## 9) BFS helpers (stable across backends)

Prefer these over raw `networkx.bfs_*` calls:

```python
succ = g.successors(root)         # dict[parent] -> list[children]
pred = g.predecessors(root)       # dict[node] -> parent
pred2 = g.generic_bfs_predecessors(root)  # always generic implementation
```

---

## 10) Laplacians and matrices

### Old

```python
L = networkx.laplacian_matrix(g)
```

### New

```python
L  = g.laplacian_matrix()
NL = g.normalized_laplacian_matrix()
```

These return SciPy sparse arrays and work for both backends (with separate implementations).

---

## 11) MST wrapper

### New

```python
mst = g.minimum_spanning_tree_from_edge_weight("weight")
```

Works for both NX and RX; returns a new `Graph`.

---

## 12) JSON serialization

### Old

```python
g.to_json("g.json")
g2 = Graph.from_json("g.json")
```

### New

* `from_json` produces an **NX-backed** `Graph`.
* `to_json` only works when **NX-backed** (raises if RX-backed).

If you need JSON after RX work:

```python
nxg = g.to_networkx_graph()
Graph.from_networkx(nxg).to_json("g.json")
```

Geometry handling stays the same conceptually:

* `include_geometries_as_geojson=True` keeps geometries (big file)
* default strips geometry objects (small file)

---

## 13) Common migration pitfalls

1. **Using `g.nodes[...]` / `g.edges[...]`**

   * Replace with `node_data()` / `edge_data()`.

2. **Assuming `subgraph()` preserves IDs**

   * True in NX mode, false in RX mode; translate results back.

3. **Passing `Graph` directly into NetworkX algorithms**

   * Use `g.get_nx_graph()` or `g.to_networkx_graph()`.

4. **Edge IDs**

   * Always obtain `eid` via `get_edge_id_from_edge((u,v))` before reading/writing edge data.

---

# New convenience functions


## Construction / conversion

- `Graph.from_null_networkx()`
  - Create an empty `Graph` with an embedded empty `networkx.Graph`.

- `Graph.from_rustworkx(rx_graph)`
  - Wrap a `rustworkx.PyGraph` as a GerryChain `Graph`.

- `Graph.to_networkx_graph()`
  - Return an equivalent `networkx.Graph` (rebuilds one if RX-backed).

- `Graph.convert_from_nx_to_rx()`
  - Convert an NX-backed `Graph` to an RX-backed `Graph` (creates id maps).

- `Graph.get_nx_to_rx_node_id_map()`
  - Return dict mapping original NX `node_id -> RX node_id` (only when RX-backed).

## Backend detection / accessors

- `Graph.is_nx_graph()`
  - `True` if embedded graph is NetworkX.

- `Graph.is_rx_graph()`
  - `True` if embedded graph is RustworkX.

- `Graph.get_nx_graph()`
  - Return embedded `networkx.Graph` (NX-only).

- `Graph.get_rx_graph()`
  - Return embedded `rustworkx.PyGraph` (RX-only).

## Node / edge ID conveniences

- `Graph.node_indices` *(property)*
  - Set of node IDs (NX labels or RX integer ids).

- `Graph.edge_indices` *(property)*
  - Set of edge IDs (NX: `(u, v)` tuples; RX: integer edge indices).

- `Graph.nodes` *(property)*
  - List of node IDs (legacy-friendly).

- `Graph.edges` *(property)*
  - Set of edge endpoint tuples `(u, v)`, even when RX-backed.

- `Graph.get_edge_from_edge_id(edge_id)`
  - Convert `edge_id -> (u, v)` tuple (NX no-op; RX uses endpoints lookup).

- `Graph.get_edge_id_from_edge((u, v))`
  - Convert `(u, v) -> edge_id` (NX returns `(u, v)`; RX returns integer edge index).

## Data access (backend-agnostic replacement for `graph.nodes[...]` / `graph.edges[...]`)

- `Graph.node_data(node_id)`
  - Return the node attribute dict for `node_id` (works for NX and RX).

- `Graph.edge_data(edge_id)`
  - Return the edge attribute dict for `edge_id` (works for NX and RX).

## Traversal / structure helpers

- `Graph.neighbors(node_id)`
  - List of neighbor node IDs (NX and RX).

- `Graph.degree(node_id)`
  - Degree of `node_id` (NX and RX).

- `Graph.islands` *(property)*
  - Set of node IDs with degree 0.

- `Graph.is_directed()`
  - Always `False` (compat with generic algorithms).

- `Graph.subgraph(nodes)`
  - Return a subgraph `Graph`.
  - **Note:** in RX, node IDs are renumbered; maps are created to translate back.

- `Graph.translate_subgraph_node_ids_for_flips(flips)`
  - Translate `{subgraph_node_id: part} -> {parent_node_id: part}`.

- `Graph.translate_subgraph_node_ids_for_set_of_nodes(set_of_nodes)`
  - Translate node IDs in a set from subgraph context -> parent context.

- `Graph.subgraphs_for_connected_components()`
  - Return `list[Graph]`, one per connected component.

- `Graph.num_connected_components()`
  - Return number of connected components.

- `Graph.is_a_tree()`
  - Return whether graph is a tree (NX uses `networkx.is_tree`; RX checks edges/nodes/CCs).

## BFS conveniences

- `Graph._generic_bfs_edges(source)`
  - Generator yielding BFS edges `(parent, child)`.

- `Graph.generic_bfs_successors_generator(root_node_id)`
  - Generator yielding `(parent, [children...])` in BFS order.

- `Graph.generic_bfs_successors(root_node_id)`
  - Dict `{parent: [children...]}` using generic BFS.

- `Graph.generic_bfs_predecessors(root_node_id)`
  - Dict `{node: parent}` using generic BFS.

- `Graph.predecessors(root_node_id)`
  - Backend-aware: uses `networkx.bfs_predecessors` for NX; generic BFS for RX.

- `Graph.successors(root_node_id)`
  - Backend-aware: uses `networkx.bfs_successors` for NX; generic BFS for RX.

## Linear algebra conveniences

- `Graph.laplacian_matrix()`
  - SciPy sparse Laplacian matrix (NX or RX implementation).

- `Graph.normalized_laplacian_matrix()`
  - SciPy sparse normalized Laplacian (NX or RX implementation).

## Algorithm wrappers

- `Graph.minimum_spanning_tree_from_edge_weight(edge_weight_attribute_name)`
  - Compute MST using NX or RX backend; return as `Graph`.

## Node ID translation / labeling helpers

- `Graph.original_nx_node_id_for_internal_node_id(internal_node_id)`
  - Map internal (especially RX) node id -> original NX label.

- `Graph.original_nx_node_ids_for_set(set_of_node_ids)`
  - Bulk map set of internal ids -> original NX labels.

- `Graph.original_nx_node_ids_for_list(list_of_node_ids)`
  - Bulk map list of internal ids -> original NX labels.

- `Graph.internal_node_id_for_original_nx_node_id(original_nx_node_id)`
  - Reverse mapping: original NX label -> current internal node id.

## Compatibility dunders / passthrough

- `Graph.__len__()`
  - Number of nodes.

- `Graph.__iter__()`
  - Iterate node IDs.

- `Graph.__getitem__(...)`
  - Only for NX-backed graphs (delegates to embedded NX graph).

- `Graph.__getattr__(name)`
  - Pass-through to embedded NX/RX graph when attribute not on wrapper.
