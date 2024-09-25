<pre>
Title:        <b>Light client with Zero Knowledge Proof</b>
Description:  Introduce zkLightclient to XRPL 
Type:         Draft

Author:       <a href="mailto:zhu@ripple.com">Zhangxiang Hu (Ripple)</a>
</pre>

# Light client with Zero Knowledge Proof

## Abstract
To interact with XRPL network, a new XRPL client node must either maintains a full transaction history or 
connects to a remote server which can provide required services to the node. 
However, while maintaining a full XRPL transaction history is resource intensive and impractical for many resource-constrained devices such as smart phones and Internet of Things, 
requiring service from a remote server relies on a strong trust assumption that the server must be honest. 

To address the high resource requirement and trust issue in running an XRPL node, 
this amendment proposes XRPL light client. 
The XRPL light client maintains a list of up-to-date XRPL ledgers (i.e., XRPL block headers) and 
apply the inclusion proof to verify that a transaction or an account state is valid on XRPL. 
However, XRPL light client itself cannot automatically update its internal states to ensure that  
the XRPL ledgers in the light client are consistent with those on the mainnet. 
Therefore, we leverage Zero-knowledge Proof technique to 
efficiently synchronize the state of XRPL light client with updated state of XRPL mainnet. 
<!--
  The Abstract is a multi-sentence (short paragraph) technical summary. This should be a very terse and human-readable version of the specification section. Someone should be able to read only the abstract to get the gist of what this specification does.

  TODO: Remove this comment before submitting
-->

## 1. Overview
This design proposes XRPL light client (xClient) ecosystem which 
contains four components: XRP ledger (XRPL), XRPL full nodes, relay network, and xClient. 

- **XRP Ledger**: XRPL is an open ledger that executes all transactions and updates the states of all ledger objects. 
It relies on the XRP Ledger Consensus Protocol (LCP) to agree on a set of transactions to add to the next ledger version. 

- **XRPL Full nodes**: A full node stores a complete copy of the blockchain's transaction history, current state, and all the rules necessary to validate transactions and blocks. 
A full node does not have to participate in the XRP LCP to create new blocks. 
However, it should be able to verify a specific transaction's validity. 
In XRPL, a full node can be easily instantiated by a validator or a tracking server. 

- **Relay network**: The relay network is an abstract entity that provides the relay service between XRPL network/full nodes and xClient. 
In particular, it interacts with the XPRL network or full nodes to fetch a ledger header $b$ and obtains a validity proof $p$ of $b$. 

- **xClient**: xClient is the instantiation of XRPL light client that only maintains a list of XRPL ledger headers. 
When a new block is created on XRPL, a relayer in the relay network will obtain the new header $b$ 
along with its associated validity proof $p$ and relay $(b, p)$ to xClient. 
xClient verifies the proof and updates its state by appending the new header to the header list. 

**![xClient architecture](https://github.com/ZhangxiangHu-Ripple/XRPL-specs/blob/main/XLS-zkLightclient/xclient_arch.png)**


### 1.1. Changes to the XRPL and Rippled
This proposal does not introduce any change to XRPL infrastructure. 
If instantiating an XRPL full node with a validator or a tracking server, 
then a new RPC method `inclusionProof` should be provided by Rippled. 
This method has no effect on consensus or transaction processing, 
thus requires no amendment.

<!--
  This section is optional.

  The motivation section should include a description of any nontrivial problems the XLS solves. It should not describe how it solves those problems, unless it is not immediately obvious. It should not describe why the XLS should be made into a standard, unless it is not immediately obvious.

  With a few exceptions, external links are not necessary in this section. If you feel that a particular resource would demonstrate a compelling case for the XLS, then save it as a printer-friendly PDF, put it in the folder with this XLS, and link to that copy.

  TODO: Remove this comment before submitting
-->

## 2. Specification

### 2.1 XRP Ledger
In this proposal, XRPL mainnet has the same assumption as described in the original [whitepaper](https://ripple.com/files/ripple_consensus_whitepaper.pdf). 
A set of validators, each maintaining its own UNL, run the XRP Ledger Consensus Protocol (LCP) to achieve consensus. 
To ensure the consistency of XRPL, an [analysis](https://arxiv.org/pdf/1802.07242) show that under a general fault model, the minimum overlap of each pair of UNLs should be greater than 90%. 
This design adopt the same requirement and assume that each pair of validators’ UNL has an overlap of more than 90% (assumption 1). 

### 2.2 Full Nodes
A full node in xClient ecosystem is the node that has a complete copy of the blockchain, 
including the entire transaction history and the current state. 
Anyone can join the XRPL network and act as a full node to provide services for others. 
We propose two main functionalities of a full node: 

1. Storing the whole XRPL transaction history and keeping the current ledger state up to date. We refer the node with full ledger history since the genesis block as the archive node, and the node with partial history (e.g., most recent 1024 blocks) as the regular full node. 

2. Storing the validation messages of the most recent leader. 

2. Responding to queries from other full nodes and relayers for information about block headers and inclusion proof. 


To respond to queries about block headers and inclusion proof, 
a full node must support following RPC methods: 

- `getblockheaders`: Returns the headers of a specific XRPL ledger header or a range of ledger headers, identified by ledger height or hash. 
- `getrecentheaders`: Returns the header of the most recent XRPL ledger. 
- `getledgerproof`: Returns the validation messages of the most recent XRPL leader.  
- `gettxproof`: Returns the information of a transaction, including the inclusion proof of the transaction, identified by the transaction hash.
- `getaccountinfo`: Returns a list of transactions associated with an account, identified by the account address. 


### 2.3 Relay Network
<!--
  The Specification section should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations.

  It is recommended to follow RFC 2119 and RFC 8170. Do not remove the key word definitions if RFC 2119 and RFC 8170 are followed.

  TODO: Remove this comment before submitting
-->

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

<!--
The following is an example of how you can document new transactions, ledger entry types, and fields:

#### **`<entry name>`** ledger entry

<High level overview, explaining the object>

##### Fields

<Any explanatory text about the fields overall>

---

| Field Name        | Required?        |  JSON Type      | Internal Type     |
|-------------------|:----------------:|:---------------:|:-----------------:|
| `<field name>` | :heavy_check_mark: | `<string, number, object, array, boolean>` | `<UINT128, UINT160, UINT256, ...>` |

<Any explanatory text about specific fields. For example, if an object must contain exactly one of three fields, note that here.>

###### Flags

> | Flag Name            | Flag Value  | Description |
>|:---------------------:|:-----------:|:------------|
>| `lsf<flag name>` | `0x0001`| <flag description> |

<Any explanatory text about specific flags>

For "Internal Type", most fields should use existing types defined in the XRPL binary format's type list here: https://xrpl.org/docs/references/protocol/binary-format#type-list . If a new type must be defined, add a separate section describing the rationale for the type, its binary format, and JSON representation.

When defining transactions, please identify any potential error scenarios. If a transaction can fail with a `tec`-class result code, specify the appropriate code. Remember that tec codes are immutable ledger entries, so changing them can cause compatibility issues with older data. Additionally, as tec codes are limited in number, it's best to reuse existing codes whenever possible. While error code details may be initially vague or incomplete, they should be refined as the proposal progresses through the candidate specification process.
-->

## Rationale

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

TBD

## Backwards Compatibility

<!--

  This section is optional.

  All XLS specs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. This section must explain how the author proposes to deal with these incompatibilities. Submissions without a sufficient backwards compatibility treatise may be rejected outright.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

No backward compatibility issues found.

## Test Cases

<!--
  This section is optional.

  The Test Cases section should include expected input/output pairs, but may include a succinct set of executable tests. It should not include project build files. No new requirements may be be introduced here (meaning an implementation following only the Specification section should pass all tests here.)
  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in the folder with this XLS. External links are discouraged.

  TODO: Remove this comment before submitting
-->

## Invariants

<!--
  This section is optional, but recommended.

Invariants are fundamental rules governing a feature's behavior that must always hold true. They define the boundaries of expected behavior and the underlying assumptions of the design. If a situation violates an invariant, it can be classified as unintended behavior, aiding in bug detection and prevention. The XRP Ledger's code incorporates invariant checks to prevent transactions from executing if they would violate an invariant rule, thereby safeguarding the ledger's immutable history from erroneous or corrupted data. While the invariants specified here can be used to create invariant checks, some may be impractical to verify at runtime.

  TODO: Remove this comment before submitting

-->

## Reference Implementation

<!--
  This section is optional.

  The Reference Implementation section should include a minimal implementation that assists in understanding or implementing this specification. It should not include project build files. The reference implementation is not a replacement for the Specification section, and the proposal should still be understandable without it.
  If the reference implementation is too large to reasonably be included inline, then consider adding it as one or more files in the folder with this XLS. External links are discouraged.

  TODO: Remove this comment before submitting
-->

## Security Considerations

<!--
  All XLS documents must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks, and can be used throughout the lifecycle of the proposal. For example, include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks, and how they are being addressed. Submissions missing the "Security Considerations" section may be rejected.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

Needs discussion.

