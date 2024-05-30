# Query IPNI with Datalog

## Authors

- [Irakli Gozalishvili]

## Background

### InterPlanetary Network Indexer (IPNI)

[InterPlanetary Network Indexer (IPNI)][IPNI] is not exactly key value store but perhaps it could be (ab)used as such to unlock an open-ended extensible innovation in user space.

IPNI takes advantage of `1:n` relation that e.g. IPFS DAG root has with blocks it consists of to provide reverse lookups, that is resolve DAG root by any of it's blocks. Exploit here is that many relations can be dropped or synced simultaneously. On the other hand trying to do the reverse would be hard to scale.

### W3 DAG Index

[W3 Index] describes DAG in terms of blocks it consists of and byte ranges of blobs addressed by the [multihash]. DAG index is encoded as a [Content Archive (CAR)][CAR] link and all the block, and [IPNI] advertisement is created such that every blob and block [multihash] lookup would resolve CAR link.

This allows clients to:

1. Perform [IPNI] lookup by DAG root block to get a DAG Index address.
2. Fetch [DAG Index] to get [multihash](es) of blobs containing DAG blocks.
3. Perform [IPNI] lookup to resolve [location commitment]s per blob.
4. Fetch [blob]s and slice it up per [DAG Index] ranges to derive set DAG blocks.
5. Built DAG from the root node using blocks.

#### W3 DAG Index Example

User can create DAG Index locally and submit it for publishing to [IPNI] through [w3up] via [w3 index] protocol.

```js
{ // CAR CID of this is "bag...idx"
  "index/sharded/dag@0.1": {
    "content": { "/": "bafy..dag" }, // Removing CID prefix leads to "block..1"
    "shards": [
      [
        // blob multihash
        { "/": { "bytes": "blb...left" } },
        // sliced within the blob
        [
          [{ "/": { "bytes": "block..1"} }, 0, 128],
          [{ "/": { "bytes": "block..2"} }, 129, 256],
          [{ "/": { "bytes": "block..3"} }, 257, 384],
          [{ "/": { "bytes": "block..4"} }, 385, 512]
        ]
      ],
      [
        // blob multihash
        { "/": { "bytes": "blb...right" } },
        // sliced within the blob
        [
          [{ "/": { "bytes": "block..5"} }, 0, 128],
          [{ "/": { "bytes": "block..6"} }, 129, 256],
          [{ "/": { "bytes": "block..7"} }, 257, 384],
          [{ "/": { "bytes": "block..8"} }, 385, 512]
        ]
      ]
    ]
  }
}
```

[W3 Index] protocol implementation will derive and publish [IPNI] advertisement such that every multihash (key) would resolve to the [DAG Index] link (entity)

> ℹ️ You can lookup `entity` by a `key`

| entity      |  key          |
| ----------- | ------------- |
| `bag...idx` | `blb...left`  |
| `bag...idx` | `block..1`    |
| `bag...idx` | `block..2`    |
| `bag...idx` | `block..3`    |
| `bag...idx` | `block..4`    |
| `bag...idx` | `blb...right` |
| `bag...idx` | `block..5`    |
| `bag...idx` | `block..6`    |
| `bag...idx` | `block..7`    |
| `bag...idx` | `block..8`    |

### Design Tradeoffs

1. Client is required to fetch `bag...idx` and parse it before it can start looking for locations for the blobs.

2. Anyone could publish an IPNI advertisement that will associate `block..1` to some `who..knows` multihash which may not be a [DAG Index], yet client will only discover that after fetching and trying to parse as such.

3. Client can not query specific relations, meaning fact that `block..1` is block of the DAG and fact that `blb...left` is a blob containing DAG blocks is not captured in any way.

4. We also have not captured relation between `blob..left` and `block..1` in any way other than they both relate to `bag...idx`.

## Proposal

Leverage [IPNI] such that we could resolve `entity` that [multihash] relates to
without having to resolve and fetch [DAG Index] first.

General idea is to capture relation names in a way that it could be utilized during lookups. This also would make indexing protocol open-ended and extensible by anyone without having intermediaries like W3Up having to adopt those extensions.

To illustrate this lets derive another lookup table from the same [DAG Index][index example], however in this case we will replace `key` column with pair of `attribute` (describing relation to an `entity`) and `value` which is a [multihash] related to `entity`.

| entity      | attribute             | value       |
| ----------- | --------------------- | ------------|
| `blb..left` | `dag@0.1/shard`       | `bafy..dag` |
| `blb..left` | `dag@0.1/shard/slice` | `block..1`  |
| `blb..left` | `dag@0.1/shard/slice` | `block..2`  |
| `blb..left` | `dag@0.1/shard/slice` | `block..3`  |
| `blb..left` | `dag@0.1/shard/slice` | `block..4`  |
| `blb..right`| `dag@0.1/shard`       | `bafy..dag` |
| `blb..right`| `dag@0.1/shard/slice` | `block..5`  |
| `blb..right`| `dag@0.1/shard/slice` | `block..6`  |
| `blb..right`| `dag@0.1/shard/slice` | `block..7`  |
| `blb..right`| `dag@0.1/shard/slice` | `block..8`  |

In this variant we have virtual `key` that is derived from `attribute` and `value` components in a deterministic manner e.g. by concatenating their bytes and computing a [multihash].

This means that we could find shards of he `bafy..dag` via IPNI query that resolves `toKey(['bafy..dag', 'dag@0.1/shard'])` key.

If system drops `blob..left` will drop all the records associated with that entity as it would be a single IPNI advertisement.

Because we get shards from single query we do not need to fetch [DAG Index] to start resolving location commitments. We could start resolving them right away and we gate IPNI with something like GraphQL we could even resolve those in a single roundtrip and cache for subsequent queries.

Location commitments and blob partition info can also be indexed and advertised on [IPNI] in a very similar way.

| entity   | attribute             | value        |
| ---------|-----------------------|--------------|
| `loc..1` | `assert/location@0.2` | `blb..left` |
| `loc..2` | `assert/location@0.2` | `blb..left` |
| `bl..idx`| `blob/partition@0.2`  | `blb..left` |
| `loc..3` | `assert/location@0.2` | `blb..right`|
| `br..idx`| `blob/partition@0.2`  | `blb..right`|

### Example Query

Here are the actual steps that would be required for computing bafy..dag DAG

1. Query IPNI for the `bafy..dag` blobs `toKey(['dag@0.1/shard', 'bafy..dag'])`
    - Receive `[blb..left, blb..right]`
2. Concurrently query IPNI for
    - Blob locations `toKey(['assert/location@0.2', 'blb..left'])`
        - Fetch blob from location
    - Blob partition `toKey(['blob/partition@0.2', 'blb..left'])`
        - Fetch resolved partition index
    - Blob locations `toKey(['assert/location@0.2', 'blb..right'])`
        - Fetch blob from location
    - Blob partition `toKey(['blob/partition@0.2', 'blb..right'])`
        - Fetch resolved partition index
3. Slice up fetched blob to blocks per partition index & put together a DAG (some parts can occur concurrently as we receive blob + index for them)

Note: round-trips can be reduced to 1 if we put something in front of IPNI that can perform a join

## Relation to Datalog

In datalog facts are triples of `[entity, attribute, value]` and you e.g. run a query like  `[?blob, 'dag@0.1/shard', 'bafy..dag']` to find all `?blob`s that have `shard` relation to `bafy..dag` which (from our example) will produce

```
{blob: 'blb..left'}
{blob: 'blb..right'}
```

But you can also perform more powerful queries that perform joins e.g.

```clj
[?blob, "dag@0.1/shard", "bafy..dag"]
[?location, "assert/location@0.2", ?blob]
[?partition, "blob/partition@0.2", ?blob]
```

Which will produce results like

```clj
{blob: "blb..left", location: "loc..1", partition: "bl..idx"}
{blob: "blb..left", location: "loc..2", partition: "bl..idx"}
{blob: "blb..right", location: "loc..3", partition: "br..idx"}
```

Only constraint we would have to impose for IPNI compatibility are

1. Every clause needs to specify constant `attribute`.
   > This is because we need attribute for lookup keys derivation
1. Variable must appear in the `entity` clause before it appears in value clause.
   > This is because of the `1:n` relation that IPNI exploits.

Such a query engine can be put in front of the IPNI to provide to offer great query flexibility in a single roundtrip. Things could be optimized significantly by utilizing caching.

[Irakli Gozalishvili]:https://github.com/gozala
[IPNI]:https://github.com/ipni/specs/blob/main/IPNI.md
[W3 Index]:https://github.com/web3-storage/specs/blob/feat/w3-index/w3-index.md
[CAR]:https://ipld.io/specs/transport/car/
[multihash]:https://github.com/multiformats/multihash
[DAG Index]:#w3-dag-index
[location commitment]:https://github.com/web3-storage/content-claims#location-claim
[datomic]:https://datomic.com
[index example]:#w3-dag-index-example
