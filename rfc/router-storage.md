# Router.Storage

## Authors

- [hannah]

## Abstract

Proposes extensions to the [W3 Blob Protocol](https://github.com/web3-storage/specs/blob/main/w3-blob.md) to enable users to expess preferences about how their blobs are stored, and a suggested implementation for a W3 blob "router", a capability provider for `/space/content/add/blob` that would invoke `/service/blob/allocate` and `service/blob/accept` on different storage nodes, based on the users preferences

## W3 Blob Protocol

To enable users to express intent for how they would like data stored, an optional `storagePreferences` argument is added to the `space/content/add/blob` invocation:

```ts
type AddBlob = {
  cmd: "/space/content/add/blob"
  sub: SpaceDID
  args: {
    blob: Blob
    storagePreferances?: StoragePreferances
  }
}

// intentionally left blank for now to expand over time
type StoragePreferences = {
}

type Blob = {
  digest:   Multihash
  size:     int
}

type Multihash = bytes
type SpaceDID = string
```

Expecting that well supported preferences will evolve over time, for now we are not attempting to define specific preference properties.

## Routed implementation of `space/content/add/blob`

A blob storage router provides the `space/content/add/blob` capability by dispatching subsequent effects to separate storage nodes that provide `service/blob/allocate` and `service/blob/accept` capabilities.

The blob router manages a state of available storage nodes.

```ts
type StorageNode = {
  did: ProviderDID
  w3upEndpoint: URL // a UCAN invocation endpoint providing at minimum `service/blob/allocate` and `service/blob/accept`
  storageProperties?: StorageProperties
}

type StorageNodes = {
  [ProviderDID]: StorageNode
}

// intentionally left blank for now to expand over time
type StorageProperties = {
}
```

The blob router will probably also need a state of where things have been placed at and what stage they're at:

```ts
type BlobAllocations = { [Blob['digest']]: Set<LocationCommitment> }

```

For now, we don't define any protocols for adding and removing providers to a router. Moreover, expecting what we track about nodes will evolve over time, for now we are not attempting to define specific storage properties for now

### `space/content/add/blob` execution flow

#### `space/content/add/blob` invoked on space:

```json
{ // "/": "bafy..add"
  "cmd": "/space/content/add/blob",
  "sub": "did:key:zAlice",
  "iss": "did:key:zAlice",
  "aud": "did:web:web3.storage",
  "args": {
    "blob": {
      // multihash of the blob as byte array
      "digest": { "/": { "bytes": "mEi...sfKg" } },
      // size of the blob in bytes
      "size": 2_097_152,
    },
    "storagePreferences": {

    }
  }
}
```

Legacy invocation of `blob/add`:

```json
{ // "/": "bafy..add"
  
  "iss": "did:key:zAlice",
  "aud": "did:web:web3.storage",
  "att": [
    {
      "can": "blob/add",
      "with": "did:key:zAlice",
      "nb": {
        "blob": {
          // multihash of the blob as byte array
          "digest": { "/": { "bytes": "mEi...sfKg" } },
          // size of the blob in bytes
          "size": 2_097_152,
        },
        "storagePreferences": {
        }
      }
    }
  ]
}
```

Receipt:

```json
{ // "/": "bafy..work",
  "iss": "did:web:web3.storage",
  "aud": "did:key:zAlice",
  "cmd": "/ucan/assert",
  "sub": "did:web:web3.storage",
  "args": {
    "assert": [
      // refers to the invocation from the example
      { "/": "bafy..add" },
      // refers to the receipt corresponding to the above invocation
      {
        "out": {
          "ok": {
            // result of the add is the content (location) commitment
            // that is produced as result of "bafy..accept"
            "site": {
              "ucan/await": [
                ".out.ok.site",
                { "/": "bafy...accept" }
              ]
            }
          }
        },
        // Previously `next` was known as `fx` instead, which is
        // set of tasks to be scheduled.
        "next": [
          // 1. System attempts to allocate memory in user space for the blob.
          { // "/": "bafy...alloc",
            "cmd": "/service/blob/allocate",
            "sub": "did:web:web3.storage",
            "args": {
              // space where memory is allocated
              "space": "did:key:zAlice",
              "blob": {
                // multihash of the blob as byte array
                "digest": { "/": { "bytes": "mEi...sfKg" } },
                // size of the blob in bytes
                "size": 2_097_152,
              },
              // task that caused this invocation
              "cause": { "/": "bafy..add" }
            }
          },
          // 2. System requests user agent (or anyone really) to upload the content
          // corresponding to the blob
          // via HTTP PUT to given location.
          { // "/": "bafy...put",
            "cmd": "/http/put",
            "sub": "did:key:zMh...der", // <-- Ed299.. derived key from content multihash
            "args": {
              // pipe url from the allocation result
              "url": {
                  "ucan/await": [
                    ".out.ok.address.url",
                    { "/": "bafy...alloc" }
                  ]
              },
              // pipe headers from the allocation result 
              "headers": {
                "ucan/await": [
                  ".out.ok.address.headers",
                  { "/": "bafy...alloc" }
                ]
              },
              // body of the http request
              "body": {
                // multihash of the blob as byte array
                "digest": { "/": { "bytes": "mEi...sfKg" } },
                "size": 2_097_152
              },
            },
            "meta": {
              // archive of the principal keys
              "keys": {
                "did:key:zMh...der": { "/": "mEi...sfKg" } 
              }
            }
          },
          // 3. System will attempt to accept uploaded content that matches blob
          // multihash and size.
          { // "/": "bafy...accept",
            "cmd": "/service/blob/accept",
            "sub": "did:web:web3.storage",
            "args": {
              "space": "did:key:zAlice",
              "blob": {
                // multihash of the blob as byte array
                "content": { "/": { "bytes": "mEi...sfKg" } },
                "size": 2_097_152,
              },
              "expires": 1712903125,
              // This task is blocked on allocation
              "_put": { "ucan/await": [".out.ok", { "/": "bafy...put" }] }
            }
          }
        ]
      }
    ]
  }
}
```

Legacy receipt for `blob/add` pre-UCAN 1.0:

```json
{ // "/": "bafy..work",
  "iss": "did:web:web3.storage",
  "aud": "did:key:zAlice",
  "ran": {
    "/": "bafy..add"
  },
  "out": {
    "ok": {
      // result of the add is the content (location) commitment
      // that is produced as result of "bafy..accept"
      "site": {
        "ucan/await": [
          ".out.ok.site",
          { "/": "bafy...accept" }
        ]
      }
    }
  },
        // Previously `next` was known as `fx` instead, which is
        // set of tasks to be scheduled.
  "fx": {
    "fork": [

      // 1. System attempts to allocate memory in user space for the blob.
      { // "/": "bafy...alloc",
        "iss": "did:web:web3.storage",
        "aud": "did:web:web3.storage",
        "att": [
          {
            "can": "/web3.storage/blob/allocate",
            "with": "did:web:web3.storage",
            "nb": {
              // space where memory is allocated
              "space": "did:key:zAlice",
              "blob": {
                // multihash of the blob as byte array
                "digest": { "/": { "bytes": "mEi...sfKg" } },
                // size of the blob in bytes
                "size": 2_097_152,
              },
              // task that caused this invocation
              "cause": { "/": "bafy..add" }
            }
          }
        ]
      },
      // 2. System requests user agent (or anyone really) to upload the content
      // corresponding to the blob
      // via HTTP PUT to given location.
      { // "/": "bafy...put",
        "iss": "did:key:zMh...der",
        "aud": "did:key:zMh...der",
        "att": [
          {
            "can": "/http/put",
            "with": "did:key:zMh...der", // <-- Ed299.. derived key from content multihash
            "nb": {
              // pipe url from the allocation result
              "url": {
                  "ucan/await": [
                    ".out.ok.address.url",
                    { "/": "bafy...alloc" }
                  ]
              },
              // pipe headers from the allocation result 
              "headers": {
                "ucan/await": [
                  ".out.ok.address.headers",
                  { "/": "bafy...alloc" }
                ]
              },
              // body of the http request
              "body": {
                // multihash of the blob as byte array
                "digest": { "/": { "bytes": "mEi...sfKg" } },
                "size": 2_097_152
              },
            },
          }
        ],
        "fct": [
          // archive of the principal keys
          "keys": {
            "did:key:zMh...der": { "/": "mEi...sfKg" } 
          }
        ]
      },
    ],
  
    // 3. System will attempt to accept uploaded content that matches blob
    // multihash and size.
    "join": { // "/": "bafy...accept",
      "iss": "did:web:web3.storage",
      "aud": "did:web:web3.storage",
      "att": {
        "can": "/web3.storage/blob/accept",
        "with": "did:web:web3.storage",
        "nb": {
          "space": "did:key:zAlice",
          "blob": {
            // multihash of the blob as byte array
            "content": { "/": { "bytes": "mEi...sfKg" } },
            "size": 2_097_152,
          },
          "expires": 1712903125,
          // This task is blocked on allocation
          "_put": { "ucan/await": [".out.ok", { "/": "bafy...put" }] }
        }
      }
    }
  }
}
```

Note that while this appears identical to for 1.0 Invocation receipts, a careful inspection of the legacy receipt suggests something slightly different has happened: unlike the current flow, the return for `blob/allocate` is NOT synchronous. 

#### Router handling of `blob/allocate`:

1. The router first checks for an existing allocation and shortcuts if it exists
2. The router runs a choice function based on the storage preferences and list of nodes available:

```ts
type ChooseNode = (prefs: StoragePreferences, nodes: StorageNodes, alreadyFailed: StorageNode[]) => StorageNode
```

**TODO: should this be a choose one or a produce complete ordering? The `alreadyFailed` argument is intended to capture previous choices that failed, to remove them from the pool, on the theory choosing 1 is probably easier and more efficient**

Such a function MUST be deterministic from the input nodes and preferences. Eventually we imagine a space having a configured CID that represents a custom implementation of such a function.

Let's say the chosen storage node is:

```json
{
  "did": "did:web:node1.storage",
  "w3upEndpoint": "https://node1.storage"
}
```

3. The service invokes `blob/allocate` remotely on the chosen node:

```json
{ // "/": "bafy..add"
  "cmd": "/blob/allocate",
  "aud": "did:web:node1.storage",
  "iss": "did:web:web3.storage",
  "sub": "did:web:node1.storage",
  "args": {
    "blob": {
      // multihash of the blob as byte array
      "digest": { "/": { "bytes": "mEi...sfKg" } },
      // size of the blob in bytes
      "size": 2_097_152,
    },
    "storagePreferences": {
    }
  }
}
```

Legacy invocation:

```json
{ // "/": "bafy..add"
  "aud": "did:web:node1.storage",
  "iss": "did:web:web3.storage",
  "att": [
    {
      "can": "/web3.storage/blob/allocate",
      "with": "did:web:node1.storage",
      "nb": {
        "blob": {
          // multihash of the blob as byte array
          "digest": { "/": { "bytes": "mEi...sfKg" } },
          // size of the blob in bytes
          "size": 2_097_152,
        },
        "storagePreferences": {
        }
      }
    }
  ]
}
```

Receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:node1.storage",
  "aud": "did:web:node1.storage",
  "sub": "did:web:node1.storage",
  "cmd": "/ucan/assert",
  "args": {
    "assert": [
      // refers to the invocation from the example
      { "/": "bafy..allocate" },
      // refers to the receipt corresponding to the above invocation
      { 
        "out": {
          "ok": {
            // result of the add is the content (location) commitment
            // that is produced as result of "bafy..accept"
            "size": 2_097_152,
            "address": {
              "url": "https://node1.storage/upload/xxxa-presigned",    
              "headers": {},
              "expires": 1712903125
            }
          }
        },
        "next": []
      },
    ],
  },
}
```

Legacy receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:node1.storage",
  "ran": {
    "/": "bafyreia5tctxekbm5bmuf6tsvragyvjdiiceg5q6wghfjiqczcuevmdqcu"
  },
  "out": {
    "ok": {
      "size": 2_097_152,
      "address": {
        "url": "https://node1.storage/upload/xxxa-presigned",    
        "headers": {},
        "expires": 1712903125
      }
    }
  },
  "fx": [],
}
```

4. On success, router creates a success receipt for the original user

Receipt:
```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:web3.storage",
  "aud": "did:web:web3.storage",
  "cmd": "/ucan/assert",
  "sub": "did:web:web3.storage",
  "args": {
    "assert": [
      // refers to the invocation from the example
      { "/": "bafy..allocate" },
      // refers to the receipt corresponding to the above invocation
      { 
        "out": {
          "ok": {
            "size": 2_097_152,
            "address": {
              "url": "https://node1.storage/upload/xxxa-presigned",    
              "headers": {},
              "expires": 1712903125
            }
          }
        },
        "next": []
      },
    ],
  },
}
```

Legacy receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:web3.storage",
  "aud": "did:web:web3.storage",
  "ran": {
    "/": "bafy..allocate"
  },
  "out": {
    "ok": {
      // result of the add is the content (location) commitment
      // that is produced as result of "bafy..accept"
      "size": 2_097_152,
      "address": {
        "url": "https://node1.storage/upload/xxxa-presigned",    
        "headers": {},
        "expires": 1712903125
      }
    }
  },
  "fx": [],
}
```

The router should also store a BlobAllocate record:

```json
{
  "did": "did:web:node1.storage",
  "blob": {
    // multihash of the blob as byte array
    "digest": { "/": { "bytes": "mEi...sfKg" } },
    // size of the blob in bytes
    "size": 2_097_152,
  },
}
```

4. On fail, router adds the failed nodes to a list of "alreadyFailed", and returns to step 2.

5. If all nodes fail, return a fail receipt to the user for `blob/allocate`

#### Router handling of blob accept

After the user puts their blob to the presigned URL, they signal to the router that `blob/accept` can proceed by sending them the UCAN receipt for `http/put`(currently this is handled differently between legacy and 1.0 UCAN)

Handling of `blob/accept` on the router is as follows:

1. Lookup Storage node that received data

```js
// psuedoCode
const blobAllocation = blobAllocations[acceptInvocation.args.blob.digest.bytes]
const storageNode = storageNodes[blobAllocation.did]
```

2. Invoke `blob/accept` on storage node

```json
 // 3. System will attempt to accept uploaded content that matches blob
  // multihash and size.
  { // "/": "bafy...accept",

    "iss": "did:web:web3.storage",
    "aud": "did:web:node1.storage",
    "cmd": "/service/blob/accept",
    "sub": "did:web:web3.storage",
    "args": {
      // as best I can tell this is used to determine who is the audience for the location commitment, which should be web3.storage
      "space": "did:web:web3.storage",
      "blob": {
        // multihash of the blob as byte array
        "content": { "/": { "bytes": "mEi...sfKg" } },
        "size": 2_097_152,
      },
      "expires": 1712903125,
    }
  }
```

Legacy Invocation:

```json
{ // "/": "bafy...accept",
  "iss": "did:web:web3.storage",
  "aud": "did:web:node1.storage",
  "att": {
    "can": "/web3.storage/blob/accept",
    "with": "did:web:web3.storage",
    "nb": {
      // as best I can tell this is used to determine who is the audience for the location commitment, which should be web3.storage?
      // alternatively maybe it should be directly to space
      "space": "did:web:web3.storage",
      "blob": {
        // multihash of the blob as byte array
        "content": { "/": { "bytes": "mEi...sfKg" } },
        "size": 2_097_152,
      },
      "expires": 1712903125,
    }
  }
}
```

Receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:node1.storage",
  "aud": "did:web:web3.storage",
  "cmd": "/ucan/assert",
  "sub": "did:web:web3.storage",
  "args": {
    "assert": [
      // refers to the invocation from the example
      { "/": "bafy..accept" },
      // refers to the receipt corresponding to the above invocation
      { 
        "out": {
          "ok": {
            // result is commitment that is produced as result of "bafy..accept"
            "site": "bafy..locationCommitment"
          }
        },
        "next": []
      },
    ],
  },
}
```

Legacy receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:node1.storage",
  "aud": "did:web:web3.storage",
  "ran": {
    "/": "bafy..accept"
  },
  "out": {
    "ok": {
      // result is commitment that is produced as result of "bafy..accept"
      "site": "bafy..locationCommitment"
    }
  },
  "fx": [],
}
```

4. Router fetches location commitment, and creates new commitment for the user.

**Note: initially I thought this was a redelegation, but I think it's maybe not -- the commitment here is to a limited download URL of JUST the blob -- it's not a w3s gateway url with full download capabilities.

the "did:web:web3.storage" is maybe really a did for two seperate things:
- the web3.storage read pipeline
- the web3.storage write pipeline

As long as we're the only ones running the read pipeline, it can all go through w3s.link, but I imagine a future situation where other people want to use another read pipeline provider.

We also need to start thinking about the difference between a storage link -- i.e. a place to just download a CAR file as is with range parameters, and a gateway endpoint -- like w3s.link that serves flat files, pathing, etc -- even in an "encodingless IPFS" world, w3s.link is different that it's CDN cached and highly available **

Original:

```json
{
  "iss": "did:web:node1.storage",
  "aud": "did:web:web3.storage",
  "cmd": "/assert/location",
  "sub": "did:web:node1.storage",
  "pol": [
    // multihash must match be for the blob uploaded
    ["==", ".content", { "/": { "bytes": "mEi...sfKg" } }],
    // must be available from this url -- but for a storage node this is probably just a raw blob, not w3s.link with full gateway support
    ["==", ".url", "https://node1.storage/raw/bafy...7fi"],
    // from this range
    ["==", ".range[0]", 0],
    ["==", ".range[1]", 2_097_152],
  ],
  // does not expire
  "exp": null
}
```

Issued to user

```json
{
  "iss": "did:web:web3.storage",
  "aud": "did:key:zAlice",
  "cmd": "/assert/location",
  "sub": "did:web:web3.storage",
  "pol": [
    // multihash must match be for the blob uploaded
    ["==", ".content", { "/": { "bytes": "mEi...sfKg" } }],
    // must be available from this url -- rewritten for w3s url
    ["==", ".url", "https://w3s.link/ipfs/bafk...7fi"],
    // from this range
    ["==", ".range[0]", 0],
    ["==", ".range[1]", 2_097_152],
  ],
  // does not expire
  "exp": null
}
```

5. Finally router produces `blob/accept` receipt for the user

Receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:web3.storage",
  "aud": "did:web:web3.storage",
  "cmd": "/ucan/assert",
  "sub": "did:web:web3.storage",
  "args": {
    "assert": [
      // refers to the invocation from the example
      { "/": "bafy..accept" },
      // refers to the receipt corresponding to the above invocation
      { 
        "out": {
          "ok": {
            // result is commitment that is produced as result of "bafy..accept"
            "site": "bafy..userLocationCommitment"
          }
        },
        "next": []
      },
    ],
  },
}
```

Legacy receipt:

```json
{ 
  // "/": "bafy..work",
  "iss": "did:web:web3.storage",
  "aud": "did:web:web3.storage",
  "ran": {
    "/": "bafy..accept"
  },
  "out": {
    "ok": {
      // result is commitment that is produced as result of "bafy..accept"
      "site": "bafy..userLocationCommitment"
    }
  },
  "fx": [],
}
```
