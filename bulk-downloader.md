# dotstorage bulk downloader

## Context

> Provide a bulk downloader for nft.storage https://github.com/w3s-project/project-tracking/issues/77

### Timeline

Last day of June

### Suggested end solutions

There are different options how the solution could look like:

1. users just receive a download link
2. users can still login into the console and request bulk download
3. or any other idea

## Built tools partially related

### w3 migration tool

- https://github.com/web3-storage/migrate-to-w3up
- Enables [web3.storage](http://web3.storage) users from [old.web3.storage](http://old.web3.storage) to migrate their uploads to a new account in w3up
    - Given [web3.storage](http://web3.storage) already hosts users’ data, this tool can follow an efficient solution of not needing to download and re-upload all the data to the new service.
    - In fact, service will figure out that it already stores the data out of the box when a user asks to store some CID. As a result, this tool simply goes through all the uploads in a [old.web3.storage](http://old.web3.storage) account and performs `store/add` invocations to make content associated with the new space for the user in w3up. Given content will be already stored, no bytes need to be wired.

## Risks and limitations with a bulk downloader as a API

The suggested end solutions above assume that an API is provided for this bulk download, i.e. all the work is done within service backend. It is important to consider the risks and limitations of such solution architecture.

This project targets a completely different goal than w3up migration tool. Here the goal would be to actually download the entire content previously uploaded by an account. This solution is technically challenging, as well as expensive for the service operator. In addition, it comes at a cost of losing these users. 

Considering the initial plan to provide a bulk download URL, there are a set of drawbacks and concerns that should be taken into account, specially when users have hundreds of GBs on their accounts:

- Not fault tolerant to errors
    - network errors no recoverable
    - no more space on disk
- Not verifiable
    - we would not be served content addressed data that can be verified, but a compacted file with all the data
- Expensive
    - not fault tolerant will mean more retries and bytes moved around over and over again on failures
    - performing compute on data at rest and concatenate it together will increase costs compared to serve it as is
    - not opportunity to optimize costs like rely on roundabout to serve raw CARs with no egress costs
- Not user friendly
    - Pause/resume later
    - Skip some files

Considering all the drawbacks of using/building/operating a solution offering a HTTP API, it is highly recommended that a simple (one click) solution is provided to migrate to w3up instead, given it is way easier to implement, cheaper to migrate and increases adoption on w3up (so business advantage). For users who prefer to opt out from this migration, we can SHOULD provide a bulk download tool where MOST work actually happens on the client side. Given the nature of this tool, it will be more complex to setup by the user, as well as take some time to download all their data.

## Proposed solution - bulk downloader on the client

A migration CLI that a [nft.storage](http://nft.storage) user can rely to bulk download their content with a Environment configuration file to have a basic configuration like an API token to access account list of files, and list of CIDs to skip from bulk download.

```jsx
NFT_STORAGE_API_TOKEN=""
SKIP_CIDS="[]"
```

The CLI process should get the list of uploads a user have and attempt to download CAR files for every single entry. By downloading CAR files instead of downloading content via root CID of the content, we can guarantee that the provided bytes match the CID of the CAR by simply hashing the bytes on the client. Moreover, while downloading all files within the user account, the tool should keep some state to enable not only start and resume, but also to continue if some sporadic failure may happen.

[nft.storage](http://nft.storage) was a project with a lot of early users, heavily under development and with multiple write interfaces with different characteristics. As a consequence, some older content MAY have less information that more recent content. For instance, when nft.storage transitioned from IPFS Cluster to S3/R2, backup URLs were introduced with information about CAR CIDs. However, old content does not have this information. In addition, the pinning API also does not have a direct relationship with the CARs that were generated with the data in IPFS Cluster days, but also not with Pick up. Considering all this, there are different ways to approach downloading content depending on their state:

- If there are backup URLs for given upload, rely on them to download the CAR with either Roundabout or Graph API (probably Roundabout with fallback to Graph API if there are rate limit errors)
- If there are no backup URLs for given upload, rely on gateway to try to retrieve content (with format CAR), probably with fallback to read from Hoverboard with bitswap if it consistently times out with the gateway?
- If it is some content from Pinning API, rely on an API on top of `complete/*` folders of S3 to attempt to download the CAR file.

Some files may not be retrievable, as for instance they may be flagged as bad content. 

When processing ends, a report file (probably based on the state written over time) should be provided. It includes files that may have failed, and reason for their failures. It also allows for a resume of the processing with this file.

## To implement

- add backup URLs to the list API (like was done for web3.storage)
- API endpoint to read from complete folders in S3 for buckets (`dotstorage-prod-0` and `dotstorage-prod-1` )
- CLI Tool

## Notes for implementation

