# Context Free Infra

## Background

Every time I have to do deal with [w3infra] I get overwhelemed by immense context passed across capability providers. There is a significant amount of bootstrap code and a lot of configuration settings. While it configuration offers flexibility, it also adds a lot of complexity which can be a maintanance burden.

## Proposal

Desipite large context, list of primitives used is pretty short:

1. Object Store (s3 / r2)
2. Data Index (Dynamo DB / s3 / r2)
3. Queues (SQS, Kenisis)
4. Email

We could greatly simplify our infrastructure if we simply exposed APIs given access to the listed primitives in terms of privileged ucan capabilities, that is capabilities that only server authorized entity can invoke. It is worth calling out that all of these primitives provide scoping mechanims object stores have buckets, indexes have tables, queues have groups or named streams.

> ℹ️ Note that scoping mechanism can also be expressed via arguments and remove a need for instantiating varios clients etc... 

To be very concrete I propose that we define `w3infra/*` capability for interacting with primitives we use and then define `w3up` service that does everything vai those capabilities without any extra context.

## Benefits

- This would significantly remove coordination needed between w3up and w3infra as infra would rearly need to change and would act as runtime native bindings.
- It would significantly simplify testing because infra will test generic APIs while w3up will test implementation specific details.

## Tradeoffs

- Tradeoff here is that service will have access to whole kingdom as opposed to limiting it to a specific bucket or a table. That said it is worth noting that today still still does have full access and could do as much harm if compromised, furthermore we could impose limits at the infra layer by delegating limited capabilities.



[w3infra]:https://github.com/web3-storage/w3infra
