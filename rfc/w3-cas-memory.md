# W3 Memory Protocol

## Abstract

Protocol describes synchronization primitives that domain specific application could utilize for concurrency control.

## Introduction

Protocol describes a `Memory` as primitive for managing a mutable state and set of operations for synchronizing it across concurrent sites.

### Memory

Memory is represented using a [Prolly Tree] (excluding leaf layer perhaps) so that it's properties can be exploited for efficient synchronisation.

### Operations

#### `memory/get`

The `memory/get` capability can be used to the obtain current state of the `Memory`.

```ts
interface MemoryGet {
  In: UCAN.Invoke<{
    Subject: SpaceDID
    Do: "memory/get"
  }>
  Out: UCAN.Receipt<{
    Ok: { state: Link<Memory> }
    Error: Unauthorized
  }>
}
```

Memory of the space that has never been updated MUST be represented as the root of the prolly tree that has no leaves.

In all other instances response should return the link to the prolly tree of the requested memory space.

> ⚠️ It yet undecided which one of the following options would be best:
> 
>  1. Respond with a root CID of the prolly tree and let recepient use content claims protocol to fetch it from the network.
>  2. Respond with a CAR CID where the prolly tree is stored and let client fetch it from the network.
>  3. Extend an API so that request can tell how many layers (from top) to be included with a response.
>  
> Right now we simply say we respond with a link to the prolly tree. That way we still have a flexibility to include underlying tree blocks in the response.
> 
> Likely choice above will depend on where and how the the prolly tree will be stored.

#### `memory/edit`

The `memory/edit` capability can be used to update memory from a given state to a desired one.

```ts
interface MemoryEdit {
  In: UCAN.Invoke<{
    Subject: SpaceDID
    Do: "memory/edit"
    Args: Edit
  }>
  Out: UCAN.Receipt<{
    Ok: { state: Link<Atom> }
    Error: RebaseRequired | InvaildEdit | Unauthorized
  }>
}

interface Edit {
  [key in ToString<ProllyTreeNode>]:
    | Remove // Remove the node
    | Edit   // Edits the node
}

type Remove = null
```

Invocation MUST supply an `Edit` operation which describes how current prolly tree MUST be edited to arrive to a desired tree. Edit MUST start from the root of the tree implying that `Args` is an object with a single key corresponding to the current root of the prolly and its value is a nested `Edit` that describes which children are removed and which children are edited.

Invocation that removes the `root` MUST be considered invalid and result in `InvalidEdit` error.

> Note that once node is edited (after childer are added / removed) it will result in the different prolly tree, but given determinstic nature of the process both actors (sender of the edit and receiver) will arrive to the same outcome.

If supplied `Edit` describes changes to the state that different from the current `RebaseRequired` error is returned requiring recepient to rebase local state before it can be published.



> ---
> # 🤔
>
> Perhaps instead of `memory/get` we need something akin of the scan. Meaning you can specify the namespace range and service responds with a merkle proof. I'm bit fuzzy however in regards to if that would assume returning the proof all they way to the leaves or not.
> 
> Scan is generally super useful if we want KV but as concurency primitive it does not really makes sense.


[Prolly Tree]:https://docs.canvas.xyz/blog/2023-05-04-merklizing-the-key-value-store.html