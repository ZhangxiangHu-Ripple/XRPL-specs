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
If instantiating an XRPL full node with a validator, a tracking server, or a Clio server, 
then some new RPC methods should be provided by these servers. 
These methods have no effect on consensus or transaction processing, 
thus requires no amendment.

<!--
  This section is optional.

  The motivation section should include a description of any nontrivial problems the XLS solves. It should not describe how it solves those problems, unless it is not immediately obvious. It should not describe why the XLS should be made into a standard, unless it is not immediately obvious.

  With a few exceptions, external links are not necessary in this section. If you feel that a particular resource would demonstrate a compelling case for the XLS, then save it as a printer-friendly PDF, put it in the folder with this XLS, and link to that copy.

  TODO: Remove this comment before submitting
-->

## 2. XRPL ledger

In this proposal, XRPL mainnet has the same assumption as described in the original [whitepaper](https://ripple.com/files/ripple_consensus_whitepaper.pdf). 
A set of validators, each maintaining its own UNL, run the XRP Ledger Consensus Protocol (LCP) to achieve consensus. 
To ensure the consistency of XRPL, an [analysis](https://arxiv.org/pdf/1802.07242) show that under a general fault model, the minimum overlap of each pair of UNLs should be greater than 90%. 
This design adopts the same requirement and **assume that each pair of validators’ UNL has an overlap of more than 90% (assumption 1)**. 

## 3. Full Nodes
A full node in xClient ecosystem is the node that has a complete copy of the blockchain, 
including the entire transaction history and the current state. 
Anyone can join the XRPL network and act as a full node to provide services for others. 
We propose two main functionalities of a full node: 

1. Storing the whole XRPL transaction history and keeping the current ledger state up to date. We refer the node with full ledger history since the genesis ledger as the archive node, and the node with partial history (e.g., most recent 1024 ledgers) as the regular full node. 

2. Storing the validation messages of the most recent leader. 

3. Responding to queries from other full nodes and relayers for information about ledger headers and transactions.

To respond to queries about the information of ledger headers and inclusion proof, 
a full node must support following RPC methods.  

### 3.1 `getblockheaders`
Returns the header of a specific XRPL ledger or headers of a set of ledgers, identified by ledger hashes or heights. 

Request format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ledger_hash`|`string`|A hex string for the requested ledger 
|`ledger_index`|`number`|The ledger index of requested ledger
|`ledger_indices`|`Array of numbers`|Array of ledger indices of requested ledgers

A valid request must have exactly one of `ledger_hash`, `ledger_index`, or `ledger_indices` included. 

Response format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ledger_meta`|`Object`|The complete header metadata of requested ledger as indicated in `ledger_hash` or `ledger_index`
|`ledger_set_meta`|`Array of Objects`|The complete header metadata of requested ledgers as indicated in `ledger_indices`

The response could follow the [standard format](https://xrpl.org/docs/references/http-websocket-apis/api-conventions/response-formatting), with the result containing information about requested ledgers, or will error if 
* Any of the field type is invalid.
* `ledger_hash` does not exist. 
* XRPL does not contain the height of `ledger_index` or any height in `ledger_indices`.

### 3.2 `getrecentheaders`
Returns the header of the most recent XRPL ledger. 
Request format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ledger_param`|`String`|A string of value `validated`. Used to request the latest validated ledger.

Response format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ledger_meta`|`Object`|The complete header metadata of the latest validated ledger. 

### 3.3 `getledgerproof`
Returns the validation messages of the most recent XRPL leader.  
Request format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ledger_proof`|`String`|A string of value `validated_proof`. Used to request the validation messages of latest validated ledger.

Response format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ledger_hash`|`String`|The hash of the most recent ledger. 
|`ledger_index`|`Number`|The ledger index of the most recent ledger. 
|`proof`|`Array of Objects`|The validation messages from XRPL validators for the most recent validated ledger. 


### 3.4 `gettxproof` 
Returns the information of a transaction, including the inclusion proof of the transaction, identified by the transaction hash.

Request format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`citd`|`String`|The compact transaction identifier of the requested transaction. Must use uppercase hexadecimal only.
|`tx_hash`|`String`|The hash of the requested transaction, as hexadecimal.
|`proof_type`|`String`|Support two proof types: a normal inclusion proof (i.e., a normal Merkle proof) or a SNARK proof (i.e. a Groth16 proof)

Response format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`ctid`|`String`|The compact transaction identifier of the requested transaction. 
|`hash`|`String`|The hash of the requested transaction.
|`ledger_hash`|`String`|The hash of the ledger that contains the requested transaction.
|`ledger_meta`|`Object`|Transaction metadata, which describes the results of the transaction.
|`proof`|`Array of Objects`|A normal Merkle proof or a SNARK proof that against `ledger_hash` 

The request fails if 
* Any of the field type is invalid.
* `ctid` or `tx_hash` does not exist.
* The full node does not support the requested `proof_type`. 

### 3.5 `getaccountinfo`
Returns the current state of an account, identified by the account address. 

Request format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`account`|`String`|The requested account address. 
|`ledger_index`|`Number or String`| The ledger index of the ledger to use, or a string of value "current" to indicate the most recent ledger.   

Response format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`account_data`|`Object`|The AccountRoot ledger object of the requested account. 

The request fails if 
* Any of the field type is invalid.
* The `account` does not exist. 
* The ledge with `ledger_index` does not contain the information of `account`. 

### 3.6 `getaccounttx` 
Returns a list of transactions associated with an account, identified by the account address. 

Request format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`account`|`String`|The requested account, identified by account address. 
|`tx_type`|`String`|The specific type of requested transaction. If not specified, then returns all transactions. 
|`ledger_hash`|`String`|If specified, only returns transactions in the specific ledger.
|`ledger_index`|`Number`|If specified, only returns transactions in the specific ledger.

If `ledger_hash` or `ledger_index` is not specified, returns the transaction in the most recent ledger. 

Response format:
|Field Name|JSON Type|Description|
|-------|---------|---------|
|`account`|`String`|The requested account, identified by account address. 
|`ledger_hash`|`String`|The ledger hash of the searched ledger.
|`ledger_index`|`Number`|The ledger index of the searched ledger.
|`tx_list`|`Array of Objects`|Array of complete transaction metadata associated with `account`. 

## 4. Relay Network
The xClient ecosystem requires a relay service to forward required information from 
XRPL mainnet or full nodes to xClient for header update and transaction verification. 

1. A relay node continuously monitor XRPL new ledger headers. 
When a new ledger header is created, the node obtains the new block header information by either 
1) monitoring the XRPL mainnet (by subscribing Validations Stream) or 
2) querying full nodes with `getledgerproof` method. 
Then the node generates the header proof with **zero-knowledge proof** technique and forwards the new ledger header along with the header proof to an xClient instance. 

2. A relay node monitors xClient request queue and retrieve requests from xClient. 

3. A relay node submits requests to full nodes via full node RPC and obtains the requested information. 

4. A relay node forwards the requested information to xClient.

4. (Optional) In case of XRPL consensus mechanism change, a relayer submits the evidence of the change to xClient. 

In this proposal, a relayer is only responsible for establishing communications between XRPL mainnet/full nodes and 
does not perform any verification of transactions or block headers, thus do not need to be trusted to behave honestly. 
However, to simplify the case of DoS, 
**we assume there is at least one honest node in the relay network to 
relay headers, transactions, and the corresponding proofs (Assumption 2)**.

A relayer in xClient model is an abstract entity. 
Anyone can join the relay network to provides relay service between XRPL and xClient. 
For example, a node can act as both a full node and a relayer. 
In practice, a potential instantiation of a relayer is the witness server, as described in 
[XLS-38](https://github.com/XRPLF/XRPL-Standards/tree/d61629cf2a5ad81153c7439f74f48d90dc96e741/XLS-0038-cross-chain-bridge).

## 5. xClient
xClient is the core component of the xClient ecosystem. 
It maintains the state (i.e., the ledger headers) history of XPRL and 
enables relayers to synchronize new ledger header of XRPL mainnet. 
The main functionalities of xClient include:

1. xClient maintains the history of XRPL states. 

2. xClient updates its internal state to synchronize the latest ledger header with XRPL mainnet, under the help from a relayer. 
Specifically, xClient receives a new ledger header along with an associated ZK proof from a relayer and 
then verifies the proof against the new header. 
xClient updates its state with the new header if the proof is valid, and rejects otherwise. 

3. xClient provides the state information to users who make queries about the XRPL states. 

4. xClient verifies the validity of XRPL transactions, under the help from a relayer. {\xnote optional?}

### 5.1 Security 

This design adopts the security definition from the peer-reviewed [zkBridge](https://arxiv.org/pdf/2210.00264) paper. 
let $View_i = [lh_1,lh_2,...,lh_i]$ where $lh_i$ indicates the ledger header at height $i$, 
xClient should satisfy the following properties: 
- **Succinctness**: For each state update, xClient should take constant 𝑂(1) time to synchronize the state.
- **Liveness**: If an honest full node receives some transaction $tx$ at some round $𝑟$, 
then $tx$ must be included into the XRPL eventually. 
xClient will eventually include a ledger header $lh_k$ such that the corresponding ledger includes the transaction $tx$. 
- **Consistency**: For any round of $i$ and $j$, it must be satisfied that either $View_𝑖$ is a prefix of $View_j$, or vice versa. 

### 5.2 Header Proof
In xClient model, either full nodes or relayers should provide validity proof for new ledger headers. 
In this design, we propose **validation message-based proof** for updating ledger headers on xClient. 
In particular, the design requires an xClient instantiation maintains its own UNL. 
when a new block is created, relayers obtain the validation messages from XRPL network which 
are signed by XRPL validators, and relay the messages to xClient. 
After receiving the required validation messages, xClient verifies the signatures against the UNL it maintains. 
If a majority of the signatures are valid, then xClient accepts the new block header. 

In XRPL consensus mechanism, to avoid forking, 
the UNLs of validators should have a high degree of overlap. 
Similarly, in this design, an xClient instantiation also uses the recommended 35 validators in default to configure its UNL. 
If more than 80% validators in xClient's UNL sign a new ledger header in validation messages, 
xClient consider the new header as valid and updates its internal state accordingly. 

We also propose to leverage ZKP to achieve succinctness. 
Specifically, xClient MUST support SNARK proof (e.g., Groth16) verification for constant proof size and verification time. 
\xnote{add details of proving system?}

**![diagram](https://github.com/ZhangxiangHu-Ripple/XRPL-specs/blob/main/XLS-zkLightclient/xclient_relayer.png)**

### 5.3 Containers
This section specifies the containers in xClient.

#### 5.3.1 `metaData`
A `metadata` container includes the meta information of xClient. 

|Field Name|JSON Type|Description|
|-------|---------|---------|
|`chainID`|`String`| The associated chain name
|`proofType`|`Number`| The proof type that xClient supports: validation messages-based or state transition-based.
|`quorum`|`Number`| Required number of signatures if `proofType` is validation messages-based proof.
|`height`|`Number`| The latest ledger index that xClient maintains.
|`lastUpdateTime`|`Time`| The proof type that xClient supports.
|`expireTime`|`Time`| xClient expires if it is offline for a while.
|`headers`|`Array of Strings`| The list of XRPL ledger header.

#### 5.3.2 `consensusData`
A `consensusData` container includes the consensus information that is used to verify a new XRPL ledger header. 

|Field Name|JSON Type|Description|
|-------|---------|---------|
|`timeStamp`|`Time`| The timestamp of the last verified block.
|`consensusType`|`String`| XRPL consensus mechanism.
|`validators`|`Array of Strings`| The UNL of xClient.
|`validatorKeys`|`Array of Strings`| The public keys of accepted validators if `proofType` has the value of 0.

#### 5.3.3 `questQueue`
A `questQueue` container includes the quests that xClient requires: `update` to update the latest ledger header and `inclusionProof` to inquire the inclusion proof of a transaction. 
A user sends a task to xClient and xClient stores the task in `questQueue`. 
A relayer picks a task from `questQueue` and provide the corresponding information to complete the task. 

|Field Name|JSON Type|Description|
|-------|---------|---------|
|`quests`|`Array of String`| A queue that stores all unfinished tasks. 


### 5.4 Methods
This section defines the publicly exposed methods in xClient. 

#### 5.4.1 `ledgerUpdate`
`ledgerUpdate` takes the input of a new XRPL ledger header and a validity proof of the header. 
If the proof is valid against the new header, xClient inserts the new header in `headers` and updates its internal states accordingly. 

Example: 
```
def ledgerUpdate(Header: headerMsg, Proof: proof) {
  # check if header exists
  if is_in(headerMsg, consensusData.headers)
    return False

  # Verify is valid. Insert header, update metaData, and consensusData. 
  if proof_verify(headerMsg, proof, metaData, consensusData)
    header_insert(header, metaData)
    metaData_update(headerMsg)
    consensus_update(headerMsg)
}
```
#### 5.4.1 `getLedger`
`getLedger` takes the input of a ledger ID and returns the associated ledger header. 
The ledger ID is a unique identifier of XRPL ledgers and can be a ledger index or a ledger header. 

Example: 
```
def getLedger(ID: ledgerID) {
  assert is_valid_id(ledgerID)
  if ledger_search(ledgerID, metaData.headers, result)
    return result

  return False
}
```

#### 5.4.1 `sendQuest`
`sendQuest` takes the input of a quest from a user and insert the quest into `questQueue`

Example:
```
def sendQuest(Quest: quest) {
  # The quest can be ledger header update, inclusion proof quest, and other defined quests. 
  quest_insert(quest, questQueue)
}
```

#### 5.4.2 `pendQuest`
A relayer picks a quest from `questQueue` and xClient set the quest as pending.

Example:
```
def pendQuest(Quest: quest) {
  # Set the quest as pending. Will be set to free after a time period. 
  assert is_valid_quest(quest)
  pend(quest)
  setExpire(time)
}
```

#### 5.4.3 `removeQuest`
A relayer sends the requested information to complete a task. xClient removes the quest from `questQueue`.

Example: 
```
def removeQuest(Quest: quest, questMessage: msg) {
  assert is_valid_msg(msg)

  # if the quest is to update the ledger header
  if is_update(quest)
    ledgerUpdate(msg.headerMsg, msg.proof)

  # if the quest is to verify a transaction
  if is_inclusionProof(quest)
    is_valid_transaction(msg.ledgerHeader, msg.txproof)
}

```
<!-- 
```

#### 2.4.4 Containers and Helper Functions 

- **Metadata**
```
MetaData {
  chainID: string;    // indicates XRPL 
  proofType: uint32;   // we propose four options
  quorum: double;    // number of valid signatures for a valid block header
  latestIndex: Height;    // current height of light client
  lastUpdateTime;    // the timestamp for the last update of xClient
  expireTime;    // xClient expires if it is offline for a while
  headers: byte[] // XPRL header list

  Initialization();    // Called by the goverance layer to create a new instance of xClient. The goverance layer decides the initial MetaData, ConsensusData, and other required parameters for xClient. 
  VerifyHeaderMsg();   // verify the header message it receives. 
  UpdateMetaData();    // update `MetaData` and `ConsensusData` if VerifyHeaderMsg() returns true.
  VerifyException();   // check if XRPL has any pre-defined infrastructure change
  UpdateException();   // update the corresponding properties that are stored in MetaData and ConsensusData 
  SubmitHeaderMsg();   // allow relayers to submit block header message to xClient. 
}
```

- **requestUpdate**
```
requestUpdate{
  "id": string;   // time, block number, transaction hash, or vlidators
  "command": string;  // value of the command according to the id
  "ledgerIndex": uint64;  // the last verified header in xClient
}

```

- **headerMsg**
```
headerMsg {
  validators: byte[];    // a list of validator information
  blockIndex: Height;    // height of the latest block 
  headerHash: byte[];    // a single header or multiple headers 
  proofType: uint32;    // choose a proof type
  headerProofs: byte[];    // signatures or proofs based on the proofType
  lastTrustedLedgerIndex: uint64 or byte[32];    // height of the last trusted block
  exceptions: uint32;    // Uncommon infrastructure change of XRPL such as consensus mechanism, fork, expired xClient, etc... used to stop xClient and invoke goverance layer. Should be well defined by the goverance layer. 

```

-->

## Todo, appendix, and discussion

1. Who should we talk to for the functionality of general-purpose XRPL light client?
2. Quest, questMessage

<!--
  The Specification section should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations.

  It is recommended to follow RFC 2119 and RFC 8170. Do not remove the key word definitions if RFC 2119 and RFC 8170 are followed.

  TODO: Remove this comment before submitting
-->

<!-- The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174. -->

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

<!-- ## Rationale -->

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

<!-- TBD -->

<!-- ## Backwards Compatibility -->

<!--

  This section is optional.

  All XLS specs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. This section must explain how the author proposes to deal with these incompatibilities. Submissions without a sufficient backwards compatibility treatise may be rejected outright.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

<!-- No backward compatibility issues found.

## Test Cases -->

<!--
  This section is optional.

  The Test Cases section should include expected input/output pairs, but may include a succinct set of executable tests. It should not include project build files. No new requirements may be be introduced here (meaning an implementation following only the Specification section should pass all tests here.)
  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in the folder with this XLS. External links are discouraged.

  TODO: Remove this comment before submitting
-->

<!-- ## Invariants -->

<!--
  This section is optional, but recommended.

Invariants are fundamental rules governing a feature's behavior that must always hold true. They define the boundaries of expected behavior and the underlying assumptions of the design. If a situation violates an invariant, it can be classified as unintended behavior, aiding in bug detection and prevention. The XRP Ledger's code incorporates invariant checks to prevent transactions from executing if they would violate an invariant rule, thereby safeguarding the ledger's immutable history from erroneous or corrupted data. While the invariants specified here can be used to create invariant checks, some may be impractical to verify at runtime.

  TODO: Remove this comment before submitting

-->

<!-- ## Reference Implementation -->

<!--
  This section is optional.

  The Reference Implementation section should include a minimal implementation that assists in understanding or implementing this specification. It should not include project build files. The reference implementation is not a replacement for the Specification section, and the proposal should still be understandable without it.
  If the reference implementation is too large to reasonably be included inline, then consider adding it as one or more files in the folder with this XLS. External links are discouraged.

  TODO: Remove this comment before submitting
-->

<!-- ## Security Considerations -->

<!--
  All XLS documents must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks, and can be used throughout the lifecycle of the proposal. For example, include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks, and how they are being addressed. Submissions missing the "Security Considerations" section may be rejected.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

<!-- Needs discussion. -->

