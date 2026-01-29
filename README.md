# GerryChain Graph Migration Guide

This is a quick “what changed / what to do instead” guide for users of GerryChain's `Graph` class
who need to migrate code from versions prior to 1.0.0 that relied on `Graph` being a subclass of 
`networkx.Graph`.

---

## “Old → New” quick replacements

| Old pattern                        | New pattern                                                   |
| ---------------------------------- | ------------------------------------------------------------- |
| `g.nodes[n][k]`                    | `g.node_data(n)[k]`                                           |
| `g.edges[u, v][k]`                 | `eid = g.get_edge_id_from_edge((u,v)); g.edge_data(eid)[k]`   |
| `for n in g.nodes:`                | `for n in g:` or `for n in g.node_indices:`                   |
| `nx.alg(g)`                        | `nx.alg(g.get_nx_graph())` or `nx.alg(g.to_networkx_graph())` |
| `sub = g.subgraph(S)` (assume IDs) | `sub = g.subgraph(S)` then translate results if RX-backed     |

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

