# Merkle References

## Abstract

Inter Planetary Linked Data (IPLD) is a fundamental building block of the IPFS. IPLD Links (a.k.a CIDs) are quintessential, yet their design has drawbacks that we call out and propose to address.

## Introduction

[Content identifiers (CIDs)][CID] claim to identify data not by _where it is stored_, but rather by the _content itself_. In our observation it instead changes addressing from "where content is stored" to "how content is stored", more specifically to how data is formatted / encoded. It is a significant improvement, but it is not addressing by "what content is".

Same data may be encoded differently depending on where it is stored, different yet when it is moved over the wire. It is also likely that encoding choices may evolve over time. Encoding choice affecting data identifier is a major drawback and counter to goal of addressing by "what content is".

Identifying data by how it is encoded has various problems and we will call out major ones to illustrate motivation for this proposal. Then we will define goals for the proposed identifiers and proceed with the proposal.

### Encoding Problem

Commonly programs operate on data structures without consideration of what processor architecture program is run or how data structures are represented in memory, all of that is abstracted away. We worry about data encoding when storing it on disk or sending over the wire. In fact, we have been able to negotiate content encoding over HTTP for decades.

In IPLD data encoding format affects its identifier. For example consider following data structure

```json
{
  "message": {
    "from": "@gozala",
    "to": "@mikeal",
    "payload": "Hi"
  }
}
```

It has a `baguqeeralquuhz37rcfn4ns3fqhqytzsg3otzzwiopefvjl3rdibl5oajj2a` content identifier in [DAG-JSON] encoding, however in [DAG-CBOR] encoding it is identified as `bafyreigxswzurrj5kprt4d4ew7qar4k6kiuiykgpq4zpefofpaq5qeb4me`. We do not really identify data by what data is, but rather identify some byte representation of it.

This is a problem because if we decide to change encoding all identifiers have to change. If we have data in [DAG-JSON] and someone asks for it in [DAG-CBOR] we won't even know we have it.

### Partitioning Problem

IPLD allow externalizing sub-structures through [IPLD Link]s. Unfortunately this also affects data identifier. In previous example if we have externalized `message` field data identifier would have been different, `baguqeera7mpc5etnxh7yi4bn7eojrmmbegkfv3czapdcxizyl33f23gszrea`.

This is a problem because data partitioned for todays constraints may fail to meet constraints of tomorrow. Depending on use cases may benefit from different partitioning strategy.

### Sizing Problem

Externalized sub-structures might be coming from untrusted machines over the network, so there is the risk that time and resources spend may be wasted on getting a data that does not match identifier requested. To mitigate this risk **block size limit** of around 2MiB is imposed, that way time and resources wasted is within reason, and you would not waste hours fetching data to discover that it does not match data identified in the request.

Implication is that produced data must fit the established size limit and producer needs to device some strategy to partition it. This tends to affect the data model, a lot of time and effort goes into optimizing specific read patterns.

Problem in disguise is that data producer must foresee how data will be consumed, which is not only difficult, but practically impossible because future is ofter full of surprises and shifting constraints.

### Problem amplified

As we have established encoding affects data identifier, and so does partitioning. Sizing constraint requires partitioning amplifying it further. There are multiple dimensions of variability that affecting data identifiers. Our example data could also have `baguqeera7mpc5etnxh7yi4bn7eojrmmbegkfv3czapdcxizyl33f23gszrea` identifier if outer structure was in [DAG-CBOR] and `message` was in [DAG-JSON].

This another level of complexity, if we store data in [DAG-CBOR] and you are asking me for it in [DAG-JSON] I may not even realize I have it. But even if I knew I had requested data and I encoded it in [DAG-JSON] links still would be in [DAG-CBOR] or I would have to traverse the whole DAG to re-encode it just to hand requested piece in desired encoding.

## Goals

Now that we have outlined problems, we can enumerate what is desired to improve upon and why:

1. Data structure SHOULD be identified by the same content identifier regardless of how parts its comprised of are stored. It SHOULD NOT matter if some parts are local and other parts external. This will get us flexibility to make choices about which parts to inline vs externalize without affecting data identifier. It will give us flexibility to make different choices based on storage constraints and possibly different choices during data transfer.

1. Data structure SHOULD be identified by the same content identifier regardless of the data encoding. This will give us flexibility to choose encoding based on storage constraint and ability to negotiate content encoding during data transfer.

## Approach

At the high level we propose to identify data structures by the root of the [reference tree], which is a binary merkle tree deterministically derived from the data structure itself.

### Data Types

[IPLD data model] offers same data types as pretty much every mainstream programming language. We use same set of data types except for the [IPLD Link], which is obsolete since every piece of data can be canonically identified.

### Reference Tree

Reference tree is a binary [merkle tree] deterministically derived from the data structure. It is derived according to the following algorithm:

1. [Abstract data format], list of two or more nodes, is derived from the data structure.
1. First node, an **operator**, denotes relation the rest of nodes, **operands** form. It is kept around for the final step.
1. Operands are [merkle fold]ed into an **operand** (sub)tree.
1. **Operator** node and **operand** (sub)tree are [merkle fold]ed into a tree that represents **reference tree** of the data.

### Merkle Fold

Merkle fold is a process of deriving [BAO] inspired binary [merkle tree] from the list of leaf nodes. Parent nodes in derived tree have exactly two children and the content of each parent node is the hash of its left child concatenated with the hash of its right child. When there's an odd number of nodes in a given level of the tree, the rightmost node is raised to the level above.

```mermaid
graph TB
parent(root) --> A(leaf)
parent --> B(leaf)

parent3(root) --> parent2(node)
parent2 --> a(leaf)
parent2 --> b(leaf)
parent3 --> c(leaf)

parent4(root) --> parent5(node)
parent4 --> parent6(node)
parent5 --> ch51(leaf)
parent5 --> ch52(leaf)
parent6 --> ch61(leaf)
parent6 --> ch62(leaf)
```

Merkle fold of a single leaf is a leaf itself

```mermaid
graph TB
only(leaf)
``````

Merkle fold of zero leaves is an empty hash (hash of zero bytes)

```mermaid
graph TB
_{ }
```

### Abstract Data Format

Abstract data format is a list of two or more nodes defined for each data type individually. Each node of the abstract data format is either:

- Leaf node represented by a byte array
- [Reference tree]

#### Scalars

| Type    | Format                                     |
|---------|--------------------------------------------|
| Null    | Empty byte array                           |
| Boolean | Single byte `0` for `false`, `1 for`true`  |
| Integer | [LEB128]                                   |
| Float   | [Double-precision floating-point format]   |
| String  | [UTF-8]                                    |
| Bytes   | Raw bytes                                  |

#### Null

Null is represented as a [merkle fold] of two leaf nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:null` and describes binary encoding format of the right node. Right node is a byte array containing zero bytes.

```mermaid
stateDiagram-v2
bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq: null
[*] --> bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq: bgc…3xtq
bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq --> b3emgfhqkq6fxuygqrh4tmfyjwzoimk4acf6i4f7vqcwuw4jopmtq_bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq  : b3e…pmtq
b3emgfhqkq6fxuygqrh4tmfyjwzoimk4acf6i4f7vqcwuw4jopmtq_bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq: merkle-structureːnull
bytes_bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq:<br/>
bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq --> bytes_bgcw577yqly5wcktxtcseninyl4u3sqwzrlqmdkugxrncr67x3xtq
```

#### Boolean

Booleans is represented as a [merkle fold] two leaf nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:boolean/byte` and describes binary encoding format of the right node. Right node is a byte array containing a single byte either `1` for `true` or `0` for false.

```mermaid
stateDiagram-v2
bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa: true
[*] --> bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa: bd5…jmaa
bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa --> b3vl45yjmn2x6ggu7n52xfg4zgt4qsiwxzdosjidt2hjuhdyd35oq_bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa  : b3v…35oq
b3vl45yjmn2x6ggu7n52xfg4zgt4qsiwxzdosjidt2hjuhdyd35oq_bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa: merkle-structureːboolean/byte
bytes_bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa: 0x01
bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa --> bytes_bd5gsrluwlf2unzhgd3jidzhmwclpyohd3ccm7yqqhc4tn6fejmaa
```

```mermaid
stateDiagram-v2
bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca: false
[*] --> bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca: bl6…3gca
bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca --> b3vl45yjmn2x6ggu7n52xfg4zgt4qsiwxzdosjidt2hjuhdyd35oq_bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca  : b3v…35oq
b3vl45yjmn2x6ggu7n52xfg4zgt4qsiwxzdosjidt2hjuhdyd35oq_bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca: merkle-structureːboolean/byte
bytes_bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca: 0x00
bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca --> bytes_bl6afhktctiibopldpshfthiitlivdkvox6x4rwqakj5ubhz33gca
```

#### String

String is represented as a [merkle fold] of pair of leaf nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:string/utf-8` and describes binary encoding format of the right node. Right node is string content encoded in UTF-8 encoding.


```mermaid
stateDiagram-v2
b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq: "hello world"
[*] --> b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq: b2i…v2tq
b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq --> bpedmjxitfak5zao5jiabw4ngew6bi3mv4y2fh5li2tdaisujt2ma_b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq  : bpe…t2ma
bpedmjxitfak5zao5jiabw4ngew6bi3mv4y2fh5li2tdaisujt2ma_b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq: merkle-structureːstring/utf-8
bytes_b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq: 0x68 0x65 0x6C…0x72 0x6C 0x64
b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq --> bytes_b2ip5bcmbwyfmckglvjbttorkwz4seqyqpyq425g6iyvyf2d6v2tq
```

#### Integer

Integer is represented as a [merkle fold] of pair of leaf nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:integer/leb128` and describes binary encoding format of the right node. Right node is [LEB128] encoded integer.

```mermaid
stateDiagram-v2
b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q: 1985
[*] --> b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q: b4o…hx4q
b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q --> b3txd3gwyvlgf7oqtiijwjatz5rq2vlcznuqmsz6nmhq7skt262yq_b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q  : b3t…62yq
b3txd3gwyvlgf7oqtiijwjatz5rq2vlcznuqmsz6nmhq7skt262yq_b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q: merkle-structureːinteger/leb128
bytes_b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q: 0xC1 0x0F
b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q --> bytes_b4ob7njt6ngtc7723fryqym6uemvyvvfntjwphglwe3ytglbwhx4q
```

#### Float

Float is represented as a [merkle fold] of pair of leaf nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:float/double-precision` and describes binary encoding format of the right node. Right node is a float encoded in [double-precision floating-point format].

```mermaid
stateDiagram-v2
bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa: 18.033
[*] --> bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa: bmj…27fa
bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa --> b46btqxqak3bstb7pmf2kkbl2ht6zat2xub5lovb4oavki4uysd2q_bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa  : b46…sd2q
b46btqxqak3bstb7pmf2kkbl2ht6zat2xub5lovb4oavki4uysd2q_bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa: merkle-structureːfloat/double-precision
bytes_bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa: 0x9C 0xC4 0x20…0x08 0x32 0x40
bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa --> bytes_bmjrgvd75uynefn3hljzkl2lg4xqthymoqolc22qwtxl2crew27fa
```

#### Bytes

Bytes is represented as a [merkle fold] of pair of leaf nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:bytes/raw` and describes binary encoding format of the right node. Right node is a byte array of the bytes.

```mermaid
stateDiagram-v2
b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q: 0x01 0x02 0x03 0x04
[*] --> b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q: b65…fc5q
b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q --> bswtrjg6daexjgv3ubromjfq5kthk4ucph6dwulbotajlz5fdw6rq_b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q  : bsw…w6rq
bswtrjg6daexjgv3ubromjfq5kthk4ucph6dwulbotajlz5fdw6rq_b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q: merkle-structureːbytes/raw
bytes_b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q: 0x01 0x02 0x03 0x04
b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q --> bytes_b65rbugtff54dlisisdpkhlyhznhrzue3ulpe5nxdc5gj7fu3fc5q
```

#### List

Lists are identified by a root of the binary merkle tree where left node is a hash of the list encoding format identifier and right node is a binary merkle tree built out of the list element identifiers.

List is represented as a [merkle fold] of pair of nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:list/item/ref-tree` and describes format of the right node. Right node is a [merkle fold] of [reference tree]s derived from the list elements.

```mermaid
stateDiagram-v2
bwwooaxibglmzjgenm4fgrbcbu7tcorrm4epsn6m2imvxhqaauupa: [1, 2, 3]
[*] --> bwwooaxibglmzjgenm4fgrbcbu7tcorrm4epsn6m2imvxhqaauupa: bww…uupa
bwwooaxibglmzjgenm4fgrbcbu7tcorrm4epsn6m2imvxhqaauupa --> bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_bwwooaxibglmzjgenm4fgrbcbu7tcorrm4epsn6m2imvxhqaauupa: bc4…dhva
bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_bwwooaxibglmzjgenm4fgrbcbu7tcorrm4epsn6m2imvxhqaauupa: merkle-structureːlist/item/ref-tree
bwwooaxibglmzjgenm4fgrbcbu7tcorrm4epsn6m2imvxhqaauupa --> b64km4bdakn5zq5x5btlfnia6ve2qmdf4vym4yrhs7gmq3alkiqvq: b64…iqvq
b64km4bdakn5zq5x5btlfnia6ve2qmdf4vym4yrhs7gmq3alkiqvq: (1, 2, 3)
b64km4bdakn5zq5x5btlfnia6ve2qmdf4vym4yrhs7gmq3alkiqvq --> bfpdjqzf5ggayajbovvqfzz4yob5kesh7vqwmsjgegfhwp7uwqy3a: bfp…qy3a
bfpdjqzf5ggayajbovvqfzz4yob5kesh7vqwmsjgegfhwp7uwqy3a: (1, 2)
bltgczabyrmquahj4bkddzkonss6d4kxgjr7sydtpcupvw7dgtfta: 1
bfpdjqzf5ggayajbovvqfzz4yob5kesh7vqwmsjgegfhwp7uwqy3a --> bltgczabyrmquahj4bkddzkonss6d4kxgjr7sydtpcupvw7dgtfta: blt…tfta
bgc7ugo22pthcj2sjujuz2qzx5nxe7u2frqjmydtghi6krlxbn36q: 2
bfpdjqzf5ggayajbovvqfzz4yob5kesh7vqwmsjgegfhwp7uwqy3a --> bgc7ugo22pthcj2sjujuz2qzx5nxe7u2frqjmydtghi6krlxbn36q: bgc…n36q
byv7b4vainvdglwtu4uaenazvl73iubt3uehj2k46o7edzr3t3hea: 3
b64km4bdakn5zq5x5btlfnia6ve2qmdf4vym4yrhs7gmq3alkiqvq --> byv7b4vainvdglwtu4uaenazvl73iubt3uehj2k46o7edzr3t3hea: byv…3hea
```

Right node of the empty list is [merkle-fold] of zero elements

```mermaid
stateDiagram-v2
begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq: []
[*] --> begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq: beg…t7uq
begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq --> bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq: bc4…dhva
bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq: merkle-structureːlist/item/ref-tree
state unit_begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq <<choice>>
begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq --> unit_begdjb2jg3dj5rvpu2gcbcapmjvviqr2kabdxl6rjnhehqgsbt7uq
```

```mermaid
stateDiagram-v2
bnxhvhxestniwdvllxh5cbvjphldncqmv7f7kmnsbzqjgnfel7ozq: [hi]
[*] --> bnxhvhxestniwdvllxh5cbvjphldncqmv7f7kmnsbzqjgnfel7ozq: bnx…7ozq
bnxhvhxestniwdvllxh5cbvjphldncqmv7f7kmnsbzqjgnfel7ozq --> bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_bnxhvhxestniwdvllxh5cbvjphldncqmv7f7kmnsbzqjgnfel7ozq: bc4…dhva
bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_bnxhvhxestniwdvllxh5cbvjphldncqmv7f7kmnsbzqjgnfel7ozq: merkle-structureːlist/item/ref-tree
bkvgjhk3q5m7eoi7nbdw6gmhnws23vyk2hjtvbhikpppza5zttreq: "hi"
bnxhvhxestniwdvllxh5cbvjphldncqmv7f7kmnsbzqjgnfel7ozq --> bkvgjhk3q5m7eoi7nbdw6gmhnws23vyk2hjtvbhikpppza5zttreq: bkv…treq
```

```mermaid
stateDiagram-v2
bmnlrm2y57d5fgil7vyts2nzpghdfogmbi5bh4uc7dbafpgztpcqa: [Point, [x, 1], [y, 2]]
[*] --> bmnlrm2y57d5fgil7vyts2nzpghdfogmbi5bh4uc7dbafpgztpcqa: bmn…pcqa
bmnlrm2y57d5fgil7vyts2nzpghdfogmbi5bh4uc7dbafpgztpcqa --> bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_bmnlrm2y57d5fgil7vyts2nzpghdfogmbi5bh4uc7dbafpgztpcqa: bc4…dhva
bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_bmnlrm2y57d5fgil7vyts2nzpghdfogmbi5bh4uc7dbafpgztpcqa: merkle-structureːlist/item/ref-tree
bmnlrm2y57d5fgil7vyts2nzpghdfogmbi5bh4uc7dbafpgztpcqa --> b7hgpaf5ivrsqunsl57rotkyvpyj7d3f62h6j4wchdv4ykvpbgita: b7h…gita
b7hgpaf5ivrsqunsl57rotkyvpyj7d3f62h6j4wchdv4ykvpbgita: (Point, [x, 1], [y, 2])
b7hgpaf5ivrsqunsl57rotkyvpyj7d3f62h6j4wchdv4ykvpbgita --> bwjfnqzuno7uainfl4tzzi2mq2ejzyqf4h3o7knrbmmlhea5luwaa: bwj…uwaa
bwjfnqzuno7uainfl4tzzi2mq2ejzyqf4h3o7knrbmmlhea5luwaa: (Point, [x, 1])
baqopfzcuxg7c6w7yymk5te2e3f7rjltub6njicwzvelcxeglfo2a: "Point"
bwjfnqzuno7uainfl4tzzi2mq2ejzyqf4h3o7knrbmmlhea5luwaa --> baqopfzcuxg7c6w7yymk5te2e3f7rjltub6njicwzvelcxeglfo2a: baq…fo2a
b6kvwbhxcgdiwps2cy54qa3e25tdh6yloydu757wpybv4fi2a3dfa: [x, 1]
bwjfnqzuno7uainfl4tzzi2mq2ejzyqf4h3o7knrbmmlhea5luwaa --> b6kvwbhxcgdiwps2cy54qa3e25tdh6yloydu757wpybv4fi2a3dfa: b6k…3dfa
b6kvwbhxcgdiwps2cy54qa3e25tdh6yloydu757wpybv4fi2a3dfa --> bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_b6kvwbhxcgdiwps2cy54qa3e25tdh6yloydu757wpybv4fi2a3dfa: bc4…dhva
bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_b6kvwbhxcgdiwps2cy54qa3e25tdh6yloydu757wpybv4fi2a3dfa: merkle-structureːlist/item/ref-tree
b6kvwbhxcgdiwps2cy54qa3e25tdh6yloydu757wpybv4fi2a3dfa --> b5pfvoepwo575ppdq54utkmqhfmbdfwa6mmzx2luvbqszhlhxwpyq: b5p…wpyq
b5pfvoepwo575ppdq54utkmqhfmbdfwa6mmzx2luvbqszhlhxwpyq: (x, 1)
blhessiutlddrl7zivzhecgnnjehezvhxghlp3w24rnhfwptr62wa: "x"
b5pfvoepwo575ppdq54utkmqhfmbdfwa6mmzx2luvbqszhlhxwpyq --> blhessiutlddrl7zivzhecgnnjehezvhxghlp3w24rnhfwptr62wa: blh…62wa
bltgczabyrmquahj4bkddzkonss6d4kxgjr7sydtpcupvw7dgtfta: 1
b5pfvoepwo575ppdq54utkmqhfmbdfwa6mmzx2luvbqszhlhxwpyq --> bltgczabyrmquahj4bkddzkonss6d4kxgjr7sydtpcupvw7dgtfta: blt…tfta
beukqisxts7ujqex2pezbsedqu3ps7upqtjjq6jkcjt6o4aa5llua: [y, 2]
b7hgpaf5ivrsqunsl57rotkyvpyj7d3f62h6j4wchdv4ykvpbgita --> beukqisxts7ujqex2pezbsedqu3ps7upqtjjq6jkcjt6o4aa5llua: beu…llua
beukqisxts7ujqex2pezbsedqu3ps7upqtjjq6jkcjt6o4aa5llua --> bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_beukqisxts7ujqex2pezbsedqu3ps7upqtjjq6jkcjt6o4aa5llua: bc4…dhva
bc4ajht5l245lsmtjm4coojywcwsxfuje7spocvk5mnw3snvtdhva_beukqisxts7ujqex2pezbsedqu3ps7upqtjjq6jkcjt6o4aa5llua: merkle-structureːlist/item/ref-tree
beukqisxts7ujqex2pezbsedqu3ps7upqtjjq6jkcjt6o4aa5llua --> bxivg3v6ugzpm7bzc6ls47k2vpo3scjb5ckwizek464ruwe54ehiq: bxi…ehiq
bxivg3v6ugzpm7bzc6ls47k2vpo3scjb5ckwizek464ruwe54ehiq: (y, 2)
ba3uvrz66regqimh3ypdk5oagsriotfm4crg7z462kpcksaz3mk3q: "y"
bxivg3v6ugzpm7bzc6ls47k2vpo3scjb5ckwizek464ruwe54ehiq --> ba3uvrz66regqimh3ypdk5oagsriotfm4crg7z462kpcksaz3mk3q: ba3…mk3q
bgc7ugo22pthcj2sjujuz2qzx5nxe7u2frqjmydtghi6krlxbn36q: 2
bxivg3v6ugzpm7bzc6ls47k2vpo3scjb5ckwizek464ruwe54ehiq --> bgc7ugo22pthcj2sjujuz2qzx5nxe7u2frqjmydtghi6krlxbn36q: bgc…n36q
```

#### Map

Map is represented as a [merkle fold] of pair of nodes. Left node is the cryptographic hash of the UTF-8 encoded string `merkle-structure:map/k+v/ref-tree` and describes format of the right node. Right node is a [merkle fold] of attributes sorted by natural sort, where attribute is a [merkle fold] of the key [reference tree] and value [reference tree].

```mermaid
stateDiagram-v2
bh36wnfqmtfpzeuzjbbzgzwad2o5k24g2h45tdnzwlmu5g2zv6r5q: {message}
[*] --> bh36wnfqmtfpzeuzjbbzgzwad2o5k24g2h45tdnzwlmu5g2zv6r5q: bh3…6r5q
bh36wnfqmtfpzeuzjbbzgzwad2o5k24g2h45tdnzwlmu5g2zv6r5q --> bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bh36wnfqmtfpzeuzjbbzgzwad2o5k24g2h45tdnzwlmu5g2zv6r5q: bct…r5la
bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bh36wnfqmtfpzeuzjbbzgzwad2o5k24g2h45tdnzwlmu5g2zv6r5q: merkle-structureːmap/k+v/ref-tree
brtcu4tzavuqsg2225b3wljixstrc3ncwsv7fj2ugl6h7ru5upk3a: message → {from, payload, to}
bh36wnfqmtfpzeuzjbbzgzwad2o5k24g2h45tdnzwlmu5g2zv6r5q --> brtcu4tzavuqsg2225b3wljixstrc3ncwsv7fj2ugl6h7ru5upk3a: brt…pk3a
bfg2vsqxqsezfri672vr7rmapx4kxuliqvqsu6tadximgiiowbjtq: "message"
brtcu4tzavuqsg2225b3wljixstrc3ncwsv7fj2ugl6h7ru5upk3a --> bfg2vsqxqsezfri672vr7rmapx4kxuliqvqsu6tadximgiiowbjtq: bfg…bjtq
bqlqke2x7vzuyfnmrz76bvbjystdytqjt5qa5nk7vhanz2tgd6qta: {from, payload, to}
brtcu4tzavuqsg2225b3wljixstrc3ncwsv7fj2ugl6h7ru5upk3a --> bqlqke2x7vzuyfnmrz76bvbjystdytqjt5qa5nk7vhanz2tgd6qta: bql…6qta
bqlqke2x7vzuyfnmrz76bvbjystdytqjt5qa5nk7vhanz2tgd6qta --> bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bqlqke2x7vzuyfnmrz76bvbjystdytqjt5qa5nk7vhanz2tgd6qta: bct…r5la
bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bqlqke2x7vzuyfnmrz76bvbjystdytqjt5qa5nk7vhanz2tgd6qta: merkle-structureːmap/k+v/ref-tree
bqlqke2x7vzuyfnmrz76bvbjystdytqjt5qa5nk7vhanz2tgd6qta --> bnwwigko5rok3gdvfoutercs5odwjg7fowmctmg2mzvj4ys6hae2q: bnw…ae2q
bnwwigko5rok3gdvfoutercs5odwjg7fowmctmg2mzvj4ys6hae2q: (from, payload, to)
bnwwigko5rok3gdvfoutercs5odwjg7fowmctmg2mzvj4ys6hae2q --> b7333nh7pbgoqqgiwhjhfi5awdarndrie7yqc3bvpgpeam6cpwb5a: b73…wb5a
b7333nh7pbgoqqgiwhjhfi5awdarndrie7yqc3bvpgpeam6cpwb5a: (from, payload)
bschspxtqysrtju3vjos2qyjksx5btg7i5b3qtpykk6rdoqqijolq: from → "gozala"
b7333nh7pbgoqqgiwhjhfi5awdarndrie7yqc3bvpgpeam6cpwb5a --> bschspxtqysrtju3vjos2qyjksx5btg7i5b3qtpykk6rdoqqijolq: bsc…jolq
b4favdqvcabhqzapq7u342wdjiomsjkigrtwrz3kx2al5ir7rpmpa: "from"
bschspxtqysrtju3vjos2qyjksx5btg7i5b3qtpykk6rdoqqijolq --> b4favdqvcabhqzapq7u342wdjiomsjkigrtwrz3kx2al5ir7rpmpa: b4f…pmpa
b2pxmqhmw744blm6cjllkcc6pc34o4ei2k5g3k6k332k2rtrxafmq: "gozala"
bschspxtqysrtju3vjos2qyjksx5btg7i5b3qtpykk6rdoqqijolq --> b2pxmqhmw744blm6cjllkcc6pc34o4ei2k5g3k6k332k2rtrxafmq: b2p…afmq
b2w2bpvpvugjoflsgyqoq3e46pbd4ubqdtesujy2febabpvjeaqza: payload → "hi"
b7333nh7pbgoqqgiwhjhfi5awdarndrie7yqc3bvpgpeam6cpwb5a --> b2w2bpvpvugjoflsgyqoq3e46pbd4ubqdtesujy2febabpvjeaqza: b2w…aqza
byidymun6ikangmxhzxcafpq3xxwpkiawfnwobrx6qmjbalwumf6q: "payload"
b2w2bpvpvugjoflsgyqoq3e46pbd4ubqdtesujy2febabpvjeaqza --> byidymun6ikangmxhzxcafpq3xxwpkiawfnwobrx6qmjbalwumf6q: byi…mf6q
bkvgjhk3q5m7eoi7nbdw6gmhnws23vyk2hjtvbhikpppza5zttreq: "hi"
b2w2bpvpvugjoflsgyqoq3e46pbd4ubqdtesujy2febabpvjeaqza --> bkvgjhk3q5m7eoi7nbdw6gmhnws23vyk2hjtvbhikpppza5zttreq: bkv…treq
b5raywmp6ufhuu3voy24na7fwghxpgkzjpigme7gdj56ikpwvt5cq: to → "mikeal"
bnwwigko5rok3gdvfoutercs5odwjg7fowmctmg2mzvj4ys6hae2q --> b5raywmp6ufhuu3voy24na7fwghxpgkzjpigme7gdj56ikpwvt5cq: b5r…t5cq
b3l37snttnejelpinu5yf665h6gipstmwnz724cwbjilfm7lxbfrq: "to"
b5raywmp6ufhuu3voy24na7fwghxpgkzjpigme7gdj56ikpwvt5cq --> b3l37snttnejelpinu5yf665h6gipstmwnz724cwbjilfm7lxbfrq: b3l…bfrq
bdvjuzbamcoxuvnws2p2xpk6t6f5n55urxin26oa5akiakxsu7eda: "mikeal"
b5raywmp6ufhuu3voy24na7fwghxpgkzjpigme7gdj56ikpwvt5cq --> bdvjuzbamcoxuvnws2p2xpk6t6f5n55urxin26oa5akiakxsu7eda: bdv…7eda
```

Note that maps keys can also be arbitrary data structures

```mermaid
stateDiagram-v2
bxth63v735fyz67w6id63udsjv35ye6rdzbea7k4hmlj5yrcojvbq: {{x}}
[*] --> bxth63v735fyz67w6id63udsjv35ye6rdzbea7k4hmlj5yrcojvbq: bxt…jvbq
bxth63v735fyz67w6id63udsjv35ye6rdzbea7k4hmlj5yrcojvbq --> bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bxth63v735fyz67w6id63udsjv35ye6rdzbea7k4hmlj5yrcojvbq: bct…r5la
bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bxth63v735fyz67w6id63udsjv35ye6rdzbea7k4hmlj5yrcojvbq: merkle-structureːmap/k+v/ref-tree
bxztyn6j5zmwunmyvbhjwllt7k3lx2zfyy7wwr5lfuz72xby4ca6a: {x} → {y}
bxth63v735fyz67w6id63udsjv35ye6rdzbea7k4hmlj5yrcojvbq --> bxztyn6j5zmwunmyvbhjwllt7k3lx2zfyy7wwr5lfuz72xby4ca6a: bxz…ca6a
bkju7hsnqretr3ofms7vxaa27hxvfui2m3cqi3wckazneaizwfkiq: {x}
bxztyn6j5zmwunmyvbhjwllt7k3lx2zfyy7wwr5lfuz72xby4ca6a --> bkju7hsnqretr3ofms7vxaa27hxvfui2m3cqi3wckazneaizwfkiq: bkj…fkiq
bkju7hsnqretr3ofms7vxaa27hxvfui2m3cqi3wckazneaizwfkiq --> bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bkju7hsnqretr3ofms7vxaa27hxvfui2m3cqi3wckazneaizwfkiq: bct…r5la
bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_bkju7hsnqretr3ofms7vxaa27hxvfui2m3cqi3wckazneaizwfkiq: merkle-structureːmap/k+v/ref-tree
b4jvkz7x22jzroshmht5v7tfxrysswxjmqounovinczotmkzz3lhq: x → 2
bkju7hsnqretr3ofms7vxaa27hxvfui2m3cqi3wckazneaizwfkiq --> b4jvkz7x22jzroshmht5v7tfxrysswxjmqounovinczotmkzz3lhq: b4j…3lhq
blhessiutlddrl7zivzhecgnnjehezvhxghlp3w24rnhfwptr62wa: "x"
b4jvkz7x22jzroshmht5v7tfxrysswxjmqounovinczotmkzz3lhq --> blhessiutlddrl7zivzhecgnnjehezvhxghlp3w24rnhfwptr62wa: blh…62wa
bgc7ugo22pthcj2sjujuz2qzx5nxe7u2frqjmydtghi6krlxbn36q: 2
b4jvkz7x22jzroshmht5v7tfxrysswxjmqounovinczotmkzz3lhq --> bgc7ugo22pthcj2sjujuz2qzx5nxe7u2frqjmydtghi6krlxbn36q: bgc…n36q
byrk22kgqpixi76zeb2bemnul7i7vxbix6u6pe7v4k2kupbu4syra: {y}
bxztyn6j5zmwunmyvbhjwllt7k3lx2zfyy7wwr5lfuz72xby4ca6a --> byrk22kgqpixi76zeb2bemnul7i7vxbix6u6pe7v4k2kupbu4syra: byr…syra
byrk22kgqpixi76zeb2bemnul7i7vxbix6u6pe7v4k2kupbu4syra --> bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_byrk22kgqpixi76zeb2bemnul7i7vxbix6u6pe7v4k2kupbu4syra: bct…r5la
bctsusf43mtwpk26fdbuezqrxqkqfccqsojfxiop3lk33a63zr5la_byrk22kgqpixi76zeb2bemnul7i7vxbix6u6pe7v4k2kupbu4syra: merkle-structureːmap/k+v/ref-tree
b4djcedrli3x2zwp32w5ct4ssggonqc5bntazhyl66zs33nhgt2nq: y → 3
byrk22kgqpixi76zeb2bemnul7i7vxbix6u6pe7v4k2kupbu4syra --> b4djcedrli3x2zwp32w5ct4ssggonqc5bntazhyl66zs33nhgt2nq: b4d…t2nq
ba3uvrz66regqimh3ypdk5oagsriotfm4crg7z462kpcksaz3mk3q: "y"
b4djcedrli3x2zwp32w5ct4ssggonqc5bntazhyl66zs33nhgt2nq --> ba3uvrz66regqimh3ypdk5oagsriotfm4crg7z462kpcksaz3mk3q: ba3…mk3q
byv7b4vainvdglwtu4uaenazvl73iubt3uehj2k46o7edzr3t3hea: 3
b4djcedrli3x2zwp32w5ct4ssggonqc5bntazhyl66zs33nhgt2nq --> byv7b4vainvdglwtu4uaenazvl73iubt3uehj2k46o7edzr3t3hea: byv…3hea
```

### Inclusion Proofs

In the example above if we want to proove that our structure `bec..bv4a` contains string `"Hi"` we share the addresses of the siblings leading to the root. In the illustration below hashes for blue nodes constitute the proof, if we combine green node and sibling from the proof we can derive pink node and so on reaching the root.

```mermaid
graph TD

_((" ")) -- "bh3…6r5q" --> object
object["{message}"]

object -- "bct…r5la" --> obj_tag(merkle-structureːmap/k+v/ref-tree)
object -- "brt…pk3a" --> message("message → {from, payload, to}")
message -- "bfg…bjtq" --> key("'message'")
message -- "bql…6qta" --> val("{from, payload, to}")
val -- "bct…r5la" --> val_tag("merkle-structureːmap/k+v/ref-tree")
val -- "bnw…ae2q" --> val_fold("(from, payload, to)")
val_fold -- "b73…wb5a" --> left("(from, payload)")
left -- "bsc…jolq" --> from("from → 'gozala'")
left -- "b2w…aqza" --> payload("payload → 'hi'")

from -- "b4f…pmpa" --> from_k("'from'")
from -- "b2p…afmq" --> from_v("'gozala'")

payload -- "byi…mf6q" --> payload_k("'payload'")
payload -- "bkv…treq" --> payload_v("'hi'")

val_fold -- "b5r…t5cq" --> to("to → 'mikeal'")
to -- "b3l…bfrq" --> to_k("'to'")
to -- "bdv…7eda" --> to_v("'mikea'")

payload_v:::subject
payload:::derive
left:::derive
val_fold:::derive
val:::derive
message:::derive
object:::subject
payload_k:::proof
from:::proof
to:::proof
val_tag:::proof
key:::proof
obj_tag:::proof

classDef proof fill:#4dabf7,stroke-width:2px,stroke:white
classDef derive fill:#e599f7,stroke-width:2px,stroke:white
classDef subject fill:#099268
```

### Transport Flexibility

It is worth calling out that above merkle trees are just visualization of how address of the root is derived. Yet all other addresss are valid and data could be indexed by it if so desired. Put it differently granularity of the index is no longer baked it at the creation, instead it is a choice that varous actors can make when coming across the data.

We also gained flexibility in terms exchange packet sizes. Peer could ask to send around X `bytes` for `bec..bv4a`, sender can traverse the tree and pack substructures until desired packet size is met and send it over. Recepient is still able to verify that received data indeed corresponds to `bec..bv4a`.


```mermaid
graph TD

_((" ")) -- "bh3…6r5q" --> object
object["{message}"]

object -- "bct…r5la" --> obj_tag(merkle-structureːmap/k+v/ref-tree)
object -- "brt…pk3a" --> message("message → {from, payload, to}")
message -- "bfg…bjtq" --> key("'message'")
message -- "bql…6qta" --> val("{from, payload, to}")
val -- "bct…r5la" --> val_tag("merkle-structureːmap/k+v/ref-tree")
val -- "bnw…ae2q" --> val_fold("(from, payload, to)")
val_fold -- "b73…wb5a" --> left("(from, payload)")
left -- "bsc…jolq" --> from("from → 'gozala'")
left -- "b2w…aqza" --> payload("payload → 'hi'")

from -- "b4f…pmpa" --> from_k("'from'")
from -- "b2p…afmq" --> from_v("'gozala'")

payload -- "byi…mf6q" --> payload_k("'payload'")
payload -- "bkv…treq" --> payload_v("'hi'")

val_fold -- "b5r…t5cq" --> to("to → 'mikeal'")
to -- "b3l…bfrq" --> to_k("'to'")
to -- "bdv…7eda" --> to_v("'mikea'")

object:::first
message:::first
obj_tag:::first
key:::first
val:::first
val_tag:::second
val_fold:::second
left:::second
to:::second
to_k:::second
to_v:::second

classDef third fill:#4dabf7,stroke-width:2px,stroke:white
classDef second fill:#e599f7,stroke-width:2px,stroke:white
classDef first fill:#099268
```

### Codec Flexibility

We have descried how addresses for data are computed not how data sholud be encoded for storage. One could choose to store data in CBOR format, in plain JSON or whatever else. In fact when peers connect they could negotiate encoding they wish to use for the session and encode data accordingly.

Note that most addresses are never stored, or send across the wire. They are simply derived from data. Only case where you want to send addresses is when you're doing a partial sync and want to reference  subtree without having to transmit it.

[abstract data format]:#abstract-data-format
[reference tree]:#reference-tree
[merkle fold]:#merkle-fold
[reference]:https://en.wikipedia.org/wiki/Reference_(computer_science)
[null pointer]:https://en.wikipedia.org/wiki/Null_pointer
[Double-precision floating-point format]:https://en.wikipedia.org/wiki/Double-precision_floating-point_format
[UTF-8]:https://en.wikipedia.org/wiki/UTF-8
[DAG-JSON]:https://ipld.io/specs/codecs/dag-json/spec
[DAG-CBOR]:https://ipld.io/specs/codecs/dag-cbor/spec/
[IPLD Data Model]:https://ipld.io/docs/data-model/kinds/
[LEB128]:https://en.wikipedia.org/wiki/LEB128
[BAO]:https://github.com/oconnor663/bao/blob/master/docs/spec.md
[billion dollar mistake]:https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/
[tombstoning]:https://en.wikipedia.org/wiki/Tombstone_(data_store)
[Merkle tree]:https://en.wikipedia.org/wiki/Merkle_tree
[CID]:https://docs.ipfs.tech/concepts/content-addressing/
[IPLD Link]:https://ipld.io/docs/schemas/features/links/
[`Null`]:https://ipld.io/docs/data-model/kinds/#null-kind
