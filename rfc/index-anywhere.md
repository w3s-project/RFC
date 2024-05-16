# Index Anywhere

## Background

W3Up heavily leverages Dynamo DB to store various indexes about the system state which is then leveraged during it's operation. It is worth calling out that in most cases (or cases where we put enough design consideration) Dynamo DB is an index over signed [UCAN]s, put it different signed [UCAN]s represent source of truth and Dynamo is an index to facilitate efficient queries.

### Dependency on AWS

Use of Dynamo for indexing ties critical component of the system to the proprietary AWS stack. It is worth calling out we have explored other DBs like [PostgreSQL], [Cloudflare KV] and [FaunaDB] and Dynamo proved to be the best choice so far.

Despite success with Dynamo it is not practical choice for an index in a distributed network, as it introduces central point of failure (owner of the db) and drives up operational costs (of the db owner) with every query served. 

### Requirements

- Replica level [ACID] transactional guarantees
- Efficient syn
- Partial syn
- Underlying storage flexibility
- Efficient queries (comparable to what we get with Dynamo)

## Proposal

Prolly Trees provide a great foundation for efficient, potentially partial sync and were [demonstrated to be implementable on top of key value stores][prolly-tree kv]. We could implement Dynamo backend for [Okra] which would create means for replication and we could leverage some of the existing backends to replicate index in other nodes.

> ❗️ There is an assumption that Prolly Tree on top of Dynamo would be efficient enough to make added overhead non issue.

### Query System

It is important to recognize that Prolly Tree is effectively a merkelized key value store, which is far more constrained query interface than what Dynamo DB offers. More importantly, key value stores tend to impose higher upfront design costs and are less flexible to schema changes which can be deal breaker.

Luckily however graph oriented data bases like [Datomic] provide a very flexible query system, yet are [implemented on top of key value stores][Datomic interanls]. In addition it is designed such that all queries run locally, they simply sync shallow (three level deep) tree and fetch leaves containing packed groups of related records as needed and store them in local [LRU cache] for the future queries. This is a great fit because we do not want to drive system operational costs, instead we could leverage our storage layer for storing packed records and only shallow tree sync with replicas.

> ℹ️ In fact records get grouped such that related records end up in the same group which could be leveraged by domain specific applications.

Put it more simply we could put datomic inspired datalog engine on top of Prolly Trees to regain desired flexibility and efficiency of queries and derive all other requirements from the Prolly Tree itself. As an added benefit Prolly Tree could be materialized an IPLD DAG making it ecosystem native.

[PostgreSQL]:https://www.postgresql.org/
[Cloudflare KV]:https://developers.cloudflare.com/kv/
[FaunaDB]:https://fauna.com/
[ACID]:https://en.wikipedia.org/wiki/ACID
[prolly-tree kv]:https://docs.canvas.xyz/blog/2023-05-04-merklizing-the-key-value-store.html
[Okra]:https://github.com/canvasxyz/okra-js?tab=readme-ov-file#usage
[Datomic]:https://www.datomic.com/
[Datomic interanls]:https://tonsky.me/blog/unofficial-guide-to-datomic-internals/
[MRU]:https://en.wikipedia.org/wiki/Cache_replacement_policies#Most-recently-used_(MRU)
