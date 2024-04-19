# extend store/* protocol with storege/deliver and storage/confirm

## Authors

- [Vasco Santos], [Protocol Labs]

## Abstract

`store/*` protocol is missing verifiability for content actually uploaded to the provided presigned URL. This factor together with intention to create a proof of delivery makes critical the introduction of a new capabilities in the `storage/*` namespace. In this RFC, we propose `storage/deliver` and `storage/confirm` capabilities.

## Introduction

When an agent wants to store some content, agent invokes `store/add` to declare this intention. The `store/add` invocation MUST include the CID of the content to be stored. An invocation example:

```json
{
  "can": "store/add",
  "with": "did:key:abc...",
  "nb": {
    "link": "bag...",
    "size": 1234
  }
}
```

If the service is receptive to store this data, a presigned URL is returned in the receipt of `store/add` invocation, so that the client can store this content. The presigned URL includes a hash derived from the content CID to guarantee the written bytes match that CID. A receipt example:

```json
{
  "ran": "bafy...storeAdd",
  "out": {
    "ok": {
      "status": "upload",
      "url": "https://...",
      "link": "bag...",
      "allocated": 1234
    }
  },
  "fx": {
    "join": null,
    "fork": []
  },
}
```

Note that content requested to store, may already be stored. If that is the case, the service may issue receipt with `"status": "done"`.

On their schedule, the agent can post the bytes using the presigned URL. However, both the agent and the service never sign a receipt stating that this content is now stored. This is where `storage/deliver` and `storage/confirm` can help. In short, we can see `storage/deliver` as a way for the agent to sign that they did the work, and `storage/confirm` for the service to sign that the bytes were received, stored and can now be retrieved.

While not relevant for verifiability, it is also important pointing out that the service MAY need to run some data computation, issue billing events or write indexes for retrievability asynchronously. Today, web3.storage operated store service relies on "S3 Bucket events" to achieve this, but it should not be assumed that all stores web3.storage will rely on will provide such features.

## New capabilities

### `storage/deliver` capability

After the agent handled the asynchronous task of posting the bytes, the agent can invoke `storage/deliver` to notify the service that the bytes were delivered. This invocation in the future MAY include a challenge for the service based on the posted data.

```json
{
  "can": "storage/deliver",
  "with": "did:web:web3.storage",
  "nb": {
    "link": "bag...",
    "url": "https://..."
  }
}
```

### `storage/confirm` capability

When the service handles the invocation of `storage/deliver`, and potentially, after some asynchronous computations/verifications, the service invokes `storage/confirm` in order to issue a signed receipt that the flow is now finished.

```json
{
  "can": "storage/confirm",
  "with": "did:web:web3.storage",
  "nb": {
    "link": "bag...",
    "url": "https://..."
  }
}
```

## Common flows

Aiming to introduce verifiability into content storing, as well as to pave the way for a future proof of delivery, `storage/deliver` capability is proposed. It would also make services not tied with store put events to execute side effects.

To offer verifiability, `store/add` invocation should be associated with `storage/deliver` and `storage/confirm` via [effect]s. `storage/deliver` would be invoked by the client to notify the service that content was stored, and the the service should also sign a confirmation that it is true via `storage/confirm`.

### `storage/deliver` of not previously delivered content

Taking into account the above, we can look at the following flow walkthrough and diagram:

1. client invokes `store/add`

```json
{
  "can": "store/add",
  "with": "did:key:abc...",
  "nb": {
    "link": "bag...",
    "size": 1234
  }
}
```

2. service issues `store/add` receipt with presigned URL, effect for `storage/deliver` task that MAY be invoked in the future by the agent and `storage/confirm` tasks that MAY be invoked in the future by the service.

```json
{
  "ran": "bafy...storeAdd",
  "out": {
    "ok": {
      "status": "upload",
      "url": "https://...",
      "link": "bag...",
      "allocated": 1234
    }
  },
  "fx": {
    "join": { "/": "bafy...storageConfirm" },
    "fork": [{ "/": "bafy...storageDeliver" }]
  },
}
```

3. client posts the content to the provided presigned URL.

4. client invokes `storage/deliver`

```json
{
  "can": "storage/deliver",
  "with": "did:web:web3.storage",
  "nb": {
    "link": "bag...",
    "url": "https://..."
  }
}
```

5. service verifies if content was delivered by user, and queues a self invocation of `storage/deliver` that is also provided as an effect.

```json
{
  "ran": "bafy...storageDeliver",
  "out": {
    "ok": {
      "link": "bag..."
    }
  },
  "fx": {
    "join": { "/": "bafy...storageConfirm" },
    "fork": []
  },
}
```

6. service may perform some other work

7. service issues `storage/confirm` receipt

```json
{
  "ran": "bafy...storageConfirm",
  "out": {
    "ok": {
      "link": "bag..."
    }
  },
  "fx": {
    "join": null,
    "fork": []
  },
}
```

![storage-deliver-0](./storage-deliver/storage-deliver-0.svg)

### `storage/deliver` of previously delivered content

The following flow walkthrough and diagram illustrate this case:

1. client invokes `store/add`

```json
{
  "can": "store/add",
  "with": "did:key:abc...",
  "nb": {
    "link": "bag...",
    "size": 1234
  }
}
```

2. service issues `store/add` receipt with done status and effect for `storage/confirm` that may be invoked in the future by the service (or was already invoked)

```json
{
  "ran": "bafy...storeAdd",
  "out": {
    "ok": {
      "status": "done",
      "link": "bag..."
    }
  },
  "fx": {
    "join": { "/": "bafy...storageConfirm" },
    "fork": []
  },
}
```

3. service issues `storage/confirm` receipt (may have happened before 1-2, or after for a concurrent operation)

```json
{
  "ran": "bafy...storageConfirm",
  "out": {
    "ok": {
      "link": "bag..."
    }
  },
  "fx": {
    "join": null,
    "fork": []
  },
}
```

![storage-deliver-1](./storage-deliver/storage-deliver-1.svg)

### `storage/deliver` of not delivered content

The following flow walkthrough and diagram illustrate this case:

1. client invokes `store/add`

```json
{
  "can": "store/add",
  "with": "did:key:abc...",
  "nb": {
    "link": "bag...",
    "size": 1234
  }
}
```

2. service issues `store/add` receipt with presigned URL, effect for `storage/deliver` task that MAY be invoked in the future by the agent and `storage/confirm` tasks that MAY be invoked in the future by the service.

```json
{
  "ran": "bafy...storeAdd",
  "out": {
    "ok": {
      "status": "upload",
      "url": "https://...",
      "link": "bag...",
      "allocated": 1234
    }
  },
  "fx": {
    "join": { "/": "bafy...storageConfirm" },
    "fork": [{ "/": "bafy...storageDeliver" }]
  },
}
```

3. client **does not** post the content to the provided presigned URL (or request fails)

4. client invokes `storage/deliver`

```json
{
  "can": "storage/deliver",
  "with": "did:web:web3.storage",
  "nb": {
    "link": "bag...",
    "url": "https://..."
  }
}
```

5. service checks if content was delivered by user, and issues receipt stating that content was not stored

```json
{
  "ran": "bafy...storageDeliver",
  "out": {
    "error": {
      "name": "ContentNotFoundError",
      "content": { "/": "bag..." }
    }
  }
}
```

![storage-deliver-2](./storage-deliver/storage-deliver-2.svg)

## Other notes

This proposal also opens the possibility for bucket decentralization, so that we could even have Storefront to find a write target for the write. For instance, when storefront receives a `store/add` request, it could allocate it to a "Hot Storage" Saturn Node by requesting a Saturn Node for a presigned URL before returning to the client the presigned URL. Once it can get a presigned URL to write, the `store/add` receipt could also include a fork task from a task from "Hot Storage" Saturn Node, which will be performed when this node receives the content offered.

We assume this is a problem out of scope of this RFC, but worth ellaboring it. Given `store/add` may be a NOP pointing to a `storage/confirm` receipt that was already handled (i.e. when client A tries to store content `bag...a` that was previously stored by client B), a `storage/confirm` receipt will not be issued for each `store/add`, but only for the first one. In other words, store services will only run `storage/confirm` one time per CID stored. As a result, if `store` wants to do computation in this data, or create some indexes, they will always output the same result. However, billing requirements may be different. Today, they are handled at `store/add` handler level, but we could also introduce a new capability to execute code that should also run when this is the first time a given client allocated store for given content (e.g. `store/allocate`, which would be also an effect of `store/add` and would be used to track billing, etc).

[Protocol Labs]: https://protocol.ai/
[Vasco Santos]: https://github.com/vasco-santos
[effect]:https://github.com/ucan-wg/invocation/#7-effect
