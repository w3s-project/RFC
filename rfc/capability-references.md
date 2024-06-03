# Capability References

## Background

UCANs use `cmd` (`can` <=0.10 verison) field as capability identifiers. They are utilized in following ways:

1. Protocol specification is designed around those identifiers.
2. Providers route invocation to handlers bound to those identifiers.
3. Delegation hierarchies are structured around capability namespaces.

As with everything in the software we keep needing to iterate on protocol and implementations and strive to avoid breaking consumers of previous versions. It sort of works, but requires a lot of careful deliberation and there is an increasing probabality that we MAY have to adopt [Semantic Versioning][semver] at each capability, which is won't really address the problem would simply force us to deliberate. It the same time [semver] would make delegations non-forward compatible (at least not according to the current UCAN specifications).

## Proposal

Fully embracing [content addressing] may offer us an effective solution, which is also [the big idea] of union. More specifically instead of trying to version identifiers we can simply hash the interface (input to output mapping) and identify capability by it. To make it concrete consider following defintion of the [`ucan/revoke`] capability.

```ts
type UCANRevoke = Method<{
  input: Input
  output: Output
}>

type Input = {
  ucan: Link<UCAN>
  proof?: Link<UCAN>[] 
}

type Output = Result<UnixTimestamp, RevocationError>

type RevocationError = Variant<{
   UCANNotFound: { message: string }
   InvalidRevocationScope: { message: string }
   UnauthorizedRevocation: { message: string }
}>

export interface UnixTimestamp {
  time: number
}
```

Shown `UCANRevoke` could be represented with a using JSON (inspired by schema schema)

```json
{
}
```

[semver]:https://semver.org/
[content addressing]:https://docs.ipfs.tech/concepts/content-addressing/#what-is-a-cid
[the-big-idea]:https://www.unison-lang.org/docs/the-big-idea/
[`ucan/revoke`]:https://github.com/w3s-project/specs/blob/main/w3-ucan.md#revocation
