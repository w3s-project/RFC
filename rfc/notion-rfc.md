# Notion Based RFCs

> or... RFCs for people who don't git this hub

## Authors

- [Hannah Howard], [Protocol Labs]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Abstract

Allow RFCs in Notion as a backup mechanism for non-technical participants in W3S to propose RFCs

## Non-goals

1. Treating any existing notion content as RFCs
2. Migrating away from Github as the primary mechanism for RFCs

## Background

The RFC process in this repo is comfortable for programmers on the W3S team. However, W3S is not just programmers. To publish RFCs here, people need to be familiar with Github and possibly command line Git. These systems are often unfamiliar to those in non-technical roles, which makes it harder for non-technical people to publish RFCs.

This proposal is simply an minimal alternate path for preserving the ability the ability to author RFCs without Github

## Proposal

When an non-technical author wishes to publish an RFC through notion, the author SHOULD add a document to the new [W3S RFCs page](https://www.notion.so/w3sat/f8233dc100324e579cb8f21198dac929?v=78a19e5c711544dd9fe35cde4cbab5b8&pvs=4)

This will create a document based on a template basic RFC structure. The author then fills out the RFC

Once complete, the author SHOULD request review. Reviews SHOULD comment on the RFC page itself, then add their name to either the approved list or to the request changes. A reviewer MAY specify a change as blocking approval.

The W3S RFCs page is shared publicly and an author MAY request comments from an outside party.

If after a few days, there are no outstanding requests for changes, and general approval, a developer familiar with Github SHOULD export the page to markdown and add it to this repo. Submitting a pull request for the ported content is OPTIONAL.