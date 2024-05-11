# Hierarchical Capabilities

## Abstract

Currently UCAN capabilities described in [W3Up specification](https://github.com/web3-storage/specs) do not follow consistent organizational structure. To make things worse accounts on login delegate `*` capabilities and spaces delegate set of namespaces.

This creates a serious challenge for the programs (like w3up client) that attempt to infer set of methods to expose based no delegated capabilities.

We roughly two roles in the system `space` and `account` and their capability set do not overlap, so we can somewhat infer which methods to present in case of `*` ability by subject DID, but that hinges upon the fact that we use `did:key` for space and `did:mailto` for accounts. This assumbtion may be rendered obsolete in the future.

## Proposal

Re-organize capabilities into role based namespeces e.g. `/space/` and `/account/` making it possible for the program to infer which set of methods to expose. All of the `store/*` and `upload/*` etc capabilities SHOULD be remapped under `/space/` namespace.

In addition we should capture depndency across namespaces in the way capabilities are orginalizated e.g. if upload task requires storing bytes we nest `store` capabilities under `upload` here is an example layout that may work


```sh
# write permission
/space/content/add                          # upload/add
/space/content/add/shard                    # store/add
/space/content/add/shard/allocate           # store/allocate
/space/content/add/shard/write              # S3/R2 PUT

# read permission
/space/content/list                         # upload/list
/space/content/list/shard                   # store/list

# delete permission
/space/content/remove                       # upload/remove
/space/content/remove/shard                 # store/remove

# write+delete permission
/space/content/edit/add                     # upload/add
/space/content/edit/add/shard               # store/add
/space/content/edit/add/shard/allocate      # store/allocate
/space/content/edit/remove                  # upload/remove
/space/content/edit/remove/shard            # store/remove

/space/shard/add                            # store/add
/space/shard/add/allocate                   # store/allocate
```

Above organization would improve on following fronts

- Giving full access to a space would require delegating `/space/` namespace.
- Giving full access to an acocunt would required delegating `/account/` namespace.
- Agent code could derive corresponding `Account` and `Space` views methods corresponding to known abilities.
- Sharing access to `space/content/add` would allow doing all of the tasks associated with file upload (right now you need to capabilities across `store/*` and `upload/*` namespaces to do it.
- We could generate views per namespace e.g. `SpaceContent`, `SpaceShard` etc...
- Server receiving `store/add` now needs to perform `store/allocate`, to make that work we have extra logic in capability definitions so thaht capability could be derived from `store/add`. It would be a lot nicer to make those relations explicit by namespace orgination.

## Side Notes

- We could keep current capabilities and mark them deprecated, which would allow old code to remain working.
- We could greatly improve client code by avoiding having to worry about multiple namespaces.

