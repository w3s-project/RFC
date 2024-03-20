# invocation router


## Goals

- We would like to index arbitrary UCANs and allow querying those so we can iterate on [content claims protocol]


## Protocol

### Deploy provider

We can allow deploying arbitrary capability provider (implement as ucanto server) using special `provider/deploy` capability.

```iplsch
type Deploy struct {
  op  "provider/deploy"
# did of the provider
  rsc  DID 
  input Deployment
}

type Deployment union {
  UcantoService   "ucanto@6.0"
  InboxSpace     "inbox@0.1"
} representation keyed

type UcantoEndpoint struct {
 url URL
# supported capabilities
 can string[]
}

type SpaceInbox struct {
# MUST include store/add delegation used to store UCANs
  access  UCAN
# MUST be delgation from provider to did:web:web3.storage
# allowing it to issue receipts on behalf of provider 
  authority UCAN
}
```

Once provider is deployed all invocations with audience matching provider DID will be routed accordingly.

If deployed provider is `UcantoService` all matching invocations will be forwarded to that endpoint.

If provider is `InboxSpace` all matching invocations will be added to that space.


All routed invocations will be indexed by w3up and made queryable by provider authorized agents using `index/query` capability.

## Content Claims

With content claims, the `aud` is the CID, and the `iss` is the provider DID noted above.

These should be stored in the same provider space, but still indexed by `aud` (CID) and we'll allow an open query interface for CID claims this way.

We should consider any claim sent to a CID to be stored in this way (in the issuers space). In the future, we may wish to restrict or limit the actors who can publish claims to CIDs, but for now we can leave it open.


[content claims protocol]:https://hackmd.io/@gozala/content-claims