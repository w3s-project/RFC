# Networked Dot Storage

> or what does decentralization even mean?

## Authors

- [Hannah Howard], [Protocol Labs]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Abstract

This RFC puts forward a proposal for the broad parameters of "decentralization" in W3S. It defines both the goals for "decentralization" and identifies where resources should be invested first on the path to "decentralization".

## Non-goals

1. Addressing anything related to tokens
2. W3S corporate structure or nucleation plans

## Background

The current state of the world for web3.storage is as follows:

- Web3.storage UCAN protocols are open and well specified. 

- We've now created a version of these protocols that can be run from any machine-- so anyone can spin up theoretically own version of our service.

- Seperately Web3.storage backs up data to Filecoin, via open protocols.

- We are actively moving towards supporting running on top of underlying storage mechanisms that lack rich event functionality (a.k.a. bucket events)

## Why Go Further

Today, someone who is not the web3.storage team could spin up their own version of web3.storage, using the open source software we've published and the protocols we've specified. No one has done this in practice, but we are increasingly close to the point where it's plausable. Is this "sufficient decentralization"?

One proposed path forward is to not pursue "decentralization" beyond trying to make other nodes a reality, perhaps through incentivization, and leaning heavily into product improvements that differentiate our platform. We could aim for "S3 if AWS invested in open source S3 implementations and made it content addressed".

This is not the path we are going to pursue. It feels important to say that up front.

But before we go deeper into where we are headed, it's worth looking at the implied dichotomy between "decentralization" and product features. That this dichotomy is almost reflexively referred to likely results from failing to appropriately frame "decentralization" in terms of product features. So let's start by framing the product features we want to unlock through decentralization.

## The Network We Want

### No Data Loss

1. Users should be able to specify how many unique copies of their data they want backed up, and the W3S network should handle insuring that many copies remain backed up as long as the user pay to store in W3S.

### Perpetuity

1. Users should be able to store hot data in W3S that will be available for  retrieval until they stop paying for it. They should not have to worry about any single business entity holding their data disappearing. If it does, the data should go somewhere else without their needing to intervene.

### Sensible defaults with customization

1. Users should have the option to store with simply "the W3S network" rather than a specific business entity or Filecoin SP. This is a hard lesson learned from Filecoin that forcing users to choose from a marketplace where information on available options is hard to come by is a terrible user experience.

2. Users should still have the option to choose a provider they like, or to move between providers.

### Trust

In general, users should be able to "trust" W3S to handle their data well and meet service level agreements. That trust should be derived by giving users the ability to verify any actions taken on the W3S network. 

1. Users should be able to gather a detailed history of what has happened with their data when they put it on the W3S Network.

2. A user paying for storage should be able to verify that data is being stored.

3. A user paying for storage should be able to verify it can be retrieved quickly. 

4. A user paying for multiple backups should be able to verify each backup is still functioning.

5. If verification demonstrates that a particular provider is failing to provide a service that the user has paid for, the network should detect the failures and move their data automatically. (or alternatively, the user should have the ability to create a dispute and move to a different provider while getting money back)

### Distribution of resources

6. Users should not have to worry about large service providers establishing monopolies that reduce user choice or drive up costs. To insure this, W3S should emphasize data portabiltity, fairness in data distribution, and low barrier to entry for new providers.

## But who wants this?

A common refrain in response to some of the positive properties that "decentralization" might enable is "none of our existing users asked for this". 
The truth is that many of these features seem far out of reach in the current architecture of the internet. The way users have interacted with the internet, shaped by the business interests of large tech companies, has told them for three decades these features are not possible. Even so, we've already spoken to potential customers who've expressed concrete interest in this kind of product.

Content addressing provides the core from which to start building toward something better. UCAN is another key component. Filecoin is another part of the puzzle. Blockchains in general are a key component (yes trustless distributed consensus matters). Our job is to fill in the remaining missing pieces.

## Next steps

Taken as a whole, yes, this is absurdly ambitious. It IS a more complex product (especially since it's a network, not a product). Getting every property on the list above is beyond a 1 year scope. Also, W3S IS going to focus on other product features besides "decentralization", particularly those important to market segments we can get to while we build towards our end state.

But, yes, we're going to try to build this. So given that, what missing pieces SHOULD be built in the next six months? 

**This is a loosely prioritized of inituitions. It's not final and it's open to discussions.**

*Btw, we're not alone here -- because we're ultimately building a network, a critical one to the overal PLVerse vision, we can draw from other teams, and eventually, some of the required components may not be built by the W3S*

### Other implementations 

The first step on the path for W3S is indeed to build other implementations of w3up. Local.storage is our first experiment. It would be nice to produce a cloudflare only implementation. The architecture proposed in [Core Node Architecture](https://w3sat.notion.site/Filecoin-SPs-in-the-W3S-Network-16a9b66e7ef3425bbee390c6481bd752?pvs=4) is a kind of aspirational target -- an ultraportable implementation that could run in most server environments, be ultra-sandboxed, provide great update mechanisms, and support rapid iteration.

We should constinue to explore these options.

### Router.storage

We need an implementation of W3up that simply routes to other instances of W3Up. Building this implies several tasks:
- deciding who gets each piece of content
- how to manage UCAN delegations across relays

And eventually:
- how to allow nodes to join and how to decide when to kick them out

But first, we should probably just hardcode a list of nodes.

*** But isn't this a point of centralization if everyone uses it? ***

Absolutely. It's important that key coordination functions in the network like router.storage eventually operate through distributed consensus and produce verifiable results.

### Read Pipeline that finds content from many write pipelines

A retrieval service (i.e. Freeway) should not be tied to a specific instance of w3up. This implies some form of global content discovery. [W3 IPNI Indexing RFC](https://github.com/web3-storage/RFC/blob/main/rfc/ipni-w3c.md) is a major step in this direction. 

Eventually we may need some kind of regional caching.

### Repair service for both W3UP and Filecoin

Repair service means a service that creates permaneant reliable storage from a series of time-limited storage deals with providers who might fail.

It's a vital missing component today on Filecoin AND a missing component of W3S as soon as their is more than one instance. We've already encountered this problem indirectly as we've thought about deal renewals and the reality that deal terms on Filecoin do not coincide in time with users joining and leaving W3UP.

It needs to exist for Filecoin AND it needs to exist for W3S.

*** But isn't this a point of centralization if everyone uses it? ***

Absolutely. It's important that key coordination functions in the network like a repair service eventually operate through distributed consensus and produce verifiable results.

### Shared w3filecoin pipeline

W3Filecoin is an excellent service for aggregating data on to Filecoin. At the same time, running it is NOT easy. While the basic aggregation functions are simple, making Filecoin deals is not. Multiple instances of W3UP should be able to share an instance of W3Filecoin. 

*** But isn't this a point of centralization if everyone uses it? ***

Absolutely. It's important that key coordination functions in the network like w3-filecoin eventually operate through distributed consensus and produce verifiable results.

### Verification Mechanisms For Retrievals from W3UP Instances

See [Proofs Overview](https://w3sat.notion.site/Filecoin-SPs-in-the-W3S-Network-16a9b66e7ef3425bbee390c6481bd752?pvs=4)

It is important that the W3UP service level agreements are not compromised across the network by poorly performing or malicious W3Up instances.

### Filecoin.Storage

Produce an implementation of W3UP and W3Filecoin that can be run by a Filecoin L1 Storage Provider. In order words, a Filecoin storage provider should be able to be more than a backup on our network -- and eventually get seperate compensation -- if they can meet our service level requirements. See [Filecoin SPs in the W3S Network](https://w3sat.notion.site/Filecoin-SPs-in-the-W3S-Network-16a9b66e7ef3425bbee390c6481bd752?pvs=4)

## FAQ

### Is decentralization the wrong word to use?

Decentralization is both a concept so overloaded it lacks any coherence, and a narrative that our product team has validated is indeed important to many market segments in web3. Actual customers have told us they won't use our product unless it's "decentralized". Likely, in some cases this is a novel kind of CYA (cover-your-ass) -- where enterprise customers want SOC-2 compliance in a data center so they can blame someone else if data gets lost, web3 customers want services they use to be "decentralized" so they can't get called out by the internet for not being "web3". Decentralization is both an annoying meme and an important narrative we have to contend with.

I would say what we're really building is an incentivized permissionless cooperative distributed network for content addressed data. If anyone wants to try to market that, I encourage them to. But internally, maybe we should consider not saying decentralization as much. At minimum, let's align on what we're trying to achieve for our users.

### Does decentralization mean running W3S on a network like Saturn?

Saturn is an interesting experiment in crypto incentization of a retrieval network. There is a lot to learn from that experience. But there is as much to learn about what not to do as what to do. In my opinion, Saturn is over indexed on diversity of small nodes and under indexed on decentralization of coordination. Also, our revenue model with Saturn clearly needs improvement. At minimum money in needs a connection to money out.

### What about quality of service?

Web3.storage users love our reliability, ease of use, and performance. In the content addressed world, these are easily our strongest differentiators. As @alanshaw has pointed out several times, in general, we MUST maintain our level of service as we evolve the network. That is a north star. The world already has enough broken decentralized systems.

### Does "distributed consensus and produce verifiable results" mean smart contracts on a blockchain?

Most likely, yes. But I'm open to hearing about other consensus mechanisms. That said, for the specific problem of trustless consensus on limited execution of relatively small rule sets, honestly, I think blockchains are the most proven solution to date. We need to assess objectively the risks/benefits of a different approach.

### I noticed you didn't put some kind of decentralized CDN on the six month plan

Yes, this is correct. Freeway is an open source project that can be run locally, or you can use the instance we run in Cloudflare (that is widely available and cached due to Cloudflare's CDN). The most immediate goal is to support multiple storage instances. Eventually, we might build an incentivized freeway retrieval network. Or, someone else might.