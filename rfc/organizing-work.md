# Organizing work

![status:draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Alan Shaw](https://github.com/alanshaw), [Protocol Labs](https://protocol.ai/)

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Protocol Labs](https://protocol.ai/)

## Abstract

Formalize a process for working that ensures a consistent and up to date view of the work items in our pipeline. Serves to document a process that can be onboarded and observed by others.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Out of scope

* The tools or locations used to document work items. i.e. Github issues, Jira, Notion etc.
* Roadmapping/general high level direction setting.

## Pipeline

Generally a work item would move from inception to completion through a pipeline in the following order:

1. **Heap** - anyone can pile stuff onto the heap. It's just a place to track things that we may want to work on in the future. It can be of arbitrary size and detail.

    Items that are of ambiguous _certainty_ SHOULD be accompanied by an [RFC](https://github.com/web3-storage/rfc) - a PR that allows others to comment in an open and structured way on the item. For example, an RFC may be a new idea or a way of achieving a goal that could be done in multiple ways. An RFC allows others to validate the idea, question or assumption to determine with more certainty if it should be worked on.
    
    All members MAY read RFC documents and provide feedback to allow the author to determine validity of the work item and adjust or clean up if necessary.

2. **Stack** - prioritized work items. _Unlike_ a conventional stack, items are added at a position that denotes their relative priority, however, as per a conventional stack, items are usually "popped" from the top.

    Items SHOULD be moved from the heap to the stack as part of a regular lightweight planning exercise that SHOULD include members of the group that will be working on the items. At this point a short description that details the _product benefits_ and _desired end state/outcome(s)_ of the system MUST be added if not already present.

    Items on the stack SHOULD be prioritized according to the general roadmap as well as other factors, such as, but limited to, how on fire things are. Items in the stack may be re-prioritized during group planning.
    
    Multiple items in the heap MAY result in a _single_ item on the stack.

    Larger items whose outputs are an implementation of some kind MUST also result in a PR to the [specs](https://github.com/web3-storage/specs/) repo.

3. **Phase** - a short period of time where items on the stack are worked on. It focuses group effort on a particular theme and is analogous to a "sprint" in Scrum.

    A phase MUST be created by group members who will be working on the items. Members decide on a theme and select a set of items from the top of the stack to work on. At this time a size estimate SHOULD be calculated per item to determine whether enough time exists to achieve the task within the phase period.
    
    In the case where a stack item is bigger in size than a phase period, members MAY select a set of sub-tasks that can be completed in time.
    
    Items MUST be assigned a DRI (Directly Responsible Individual) _and_ a support member. The support member is primarily responsible for reviewing PRs, but MAY assist/pair in other ways. Support members may the DRI for other tasks and vice versa. Larger items MAY involve more than two group members.

    Phases SHOULD NOT last for more than one month. It is RECOMMENDED for phases to be _two weeks_ long.

    Every phase SHOULD have a name, e.g. Crusty Koala.

    It is not always possible for _all_ desired items in a phase to fit under the same umbrella/theme. It is RECOMMENDED in this case for one member of the team to be assigned to these tasks and for that member be immune from this assignment in the subsequent phase.
