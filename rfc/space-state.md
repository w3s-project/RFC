# Abstract

Currently deployed w3up protocol implementation maintains state of the (user) space in the dynamo db, which is at odds with a notion of user space as they can not alter state locally without active connection to our service.

To resolve this we need to represent state of the (user) space as DAG that user can query and edit locally and sync it with replica held by our service.

## Challenges

### Content Replication

Representing space as a [UnixFS] directory of user content or key value store like [Pail] may seem like a non-brainer, but it introduces major challenge in terms of sync, beacuse:

1. User may create entry to a content that service does not have.
2. User may add content over allocated capacity.

## Proposal

Represent state of the (user) space as [prolly tree] for the following properties:

1. Effecient sync
2. Ability to represent & sync using arbitrary key value store.
3. Ability to perform range queries

Represent entries (blob, upload, etc...) via signed service commitments. e.g. every `blob/add` would lead to `/blob/${multihash}` key mapped to receipt issued by a service for that invocation. That would allow client to list all blobs by iterating receipts and deriving their state, some entries may have receipts with effects that are still pending and some that have service location commitments.

ℹ️ It is worth calling out that while user can make arbitrary key value pairs they can not create receipts issued by the service and there for can not represent some content available without uploading and getting receipt neither they can go over storage capacity as service will not sign a receipt allowing it. This creates a system where user is completely in charge of updating state as they please, yet service is only going to recognize and uphold stored commitments it issued.

## Invocations & Receipts

It is also very tempting to make invocations and receipts part of the space state. For example we could designate key (sub)space as invocation queue. User could add any signed invocations there, during sync service will move invocations where it is an audience from the queue and place them into invoked (sub)space and add receipt into a corresponding key when invocation is complete.

User could move invocation back into queue but that will effectively be a noop as on next sync service will move it back to invoked space and write the receipt. User could also delete invocation if they want to free up some space but that should not affect system operations. 


[UnixFS]:https://github.com/ipfs/specs/blob/main/unixfs.md
[pail]:https://github.com/web3-storage/pail
[prolly tree]:https://docs.canvas.xyz/blog/2023-05-04-merklizing-the-key-value-store.html
