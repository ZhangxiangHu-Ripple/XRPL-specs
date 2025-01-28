<pre>
Title:        <b>Light client with Zero Knowledge Proof</b>
Description:  Introduce zkLightclient to XRPL 
Type:         Draft

Author:       <a href="mailto:zhu@ripple.com">Zhangxiang Hu (Ripple)</a>
</pre>

# Light client with Zero Knowledge Proof

## Abstract
To interact with XRPL network, a new XRPL client node must either maintain a full transaction history of the whole XRPL ledgers or some most recent ledgers (e.g., most recent 256 ledgers ),  
or it connects to a remote server which can provide required services to the node. 
However, while maintaining a full XRPL transaction history is resource [intensive](https://xrpl.org/docs/infrastructure/installation/capacity-planning) 
and impractical for many resource-constrained devices, such as smart phones and the Internet of Things, 
requiring service from a remote server relies on a strong trust assumption that the server must be honest. 

To address the high resource requirement and trust issue in running an XRPL node, 
this amendment proposes XRPL light client. 
The XRPL light client maintains a list of up-to-date XRPL ledgers (i.e., XRPL ledger headers) and 
apply the inclusion proof to verify that a transaction or an account state is valid on XRPL. 
However, an XRPL light client itself cannot automatically update its internal states to ensure that  
the XRPL ledgers in the light client are consistent with those on the mainnet. 
Therefore, we leverage Zero-knowledge Proof technique to 
efficiently synchronize the state of XRPL light client with the state of XRPL mainnet. 

In addition to reducing the cost of running a XRPL node, 
an XRPL light client could also be an essential building component in many applications. 
For example, in many blockchain interoperability solutions such as zkBridge, Succinct protocols, and Inter-Blockchain Communication, 
a blockchain needs to maintain a light client of the counter blockchain in order to track and efficiently verify the state of the counter blockchain. 


<!--
  The Abstract is a multi-sentence (short paragraph) technical summary. This should be a very terse and human-readable version of the specification section. Someone should be able to read only the abstract to get the gist of what this specification does.

  TODO: Remove this comment before submitting
-->

## 1. Overview
This design proposes XRPL light client (xClient) ecosystem which 
contains four components: XRP ledger (XRPL), XRPL full nodes, relay network, and xClient. 

- **XRP Ledger**: XRPL is an open ledger that executes all transactions and updates the states of all ledger objects. 
It relies on the XRP Ledger Consensus Protocol (LCP) to agree on a set of transactions to add to the next ledger version. 

- **XRPL Full nodes**: A full node stores a complete copy of the blockchain's transaction history, current state, and all the rules necessary to validate XRPL transactions and ledgers. 
A full node does not have to participate in the XRP LCP to create new ledgers. 
However, it should be able to verify a specific transaction's validity. 
In XRPL, a full node can be easily instantiated by a validator or a tracking server. 

- **Relay network**: The relay network is an abstract entity that provides the relay service between XRPL network/full nodes and xClient. 
In particular, it interacts with the XPRL network or full nodes to fetch a ledger header $b$ and obtains a validity proof $p$ of $b$. 

- **xClient**: xClient is the instantiation of XRPL light client that only maintains a list of XRPL ledger headers. 
When a new ledger is created on XRPL, a relayer in the relay network will obtain the new header $b$ 
along with its associated validity proof $p$ and relay $(b, p)$ to xClient. 
xClient verifies the proof and updates its state by appending the new header to the header list. 

```
+-----------------+
|                 |
|       XRPL      |_____ Ledger header b ____
|                 |     Validity message m   |
+-----------------+                          |
          ^                                  V
          |                        +-----------------+             +-----------------+
          |                   -----|                 |   (b, p)    |                 | -----  verifies
      Consensus      Creates |     |  Relay network  | ----------->|     xClient     |      |  (b, p)
          |          proof p  ---->|                 |             |                 |<-----  and updates
          |                        +-----------------+             +-----------------+
          V                                  ^                                      
+-----------------+                          | 
|                 |                          | 
|    Full nodes   |_____ Ledger header b ____|
|                 |     Validity message m
+-----------------+

```

An example of message flow in zkLightclient to update ledger headers in xClient is as follows: 
1. XRPL validators broadcast validation messages and generate new XRPL ledger. 
2. XRPL full nodes store the validation messages and the new ledger. 
3. A relayer fetches an updating quest from xClient's `questQueue`. 
4. A relayer subscribes to the XRPL validation messages, or queries the validation messages from full nodes for a specific ledger. 
5. The relayer receives the validation messages and extracts the messages that are signed by the nodes in xClient's UNL (the information is contained in the update quest).
6. The relayer deserialize the messages and obtains all required data fields.
7. The relayer processes the data, including the signing public key and the signature, converting them into the format that is compatible with a zero-knowledge proof (ZKP) proving system. 
8. The relayer runs a prover, generates a proof, and sends the proof along with the requested ledger header to xClient. 
This proposal suggests to use Groth16, but xClient can choose which proof it can accept. 
9. The relayer sends the reply along with the corresponding proof to xClient
10. xClient verifies the proof against the header. 
11. xClient accepts the header if the proof is valid, and rejects the header otherwise. 

Note that the message flow might be different, depends on the update modes (Section 4.3). 

```
1. Broadcast 
   messages
   ----
  |    |
  |    v
+-----------------+
|       XRPL      |
|       and       |<------4. Subscribe -------              ---------3. Quest---------- 
|    Full nodes   |        and request        |            |                          |
+-----------------+                           |            |                          |
  |    ^          |                           |            v                          |
  |    |          |                         +-----------------+             +-----------------+
   ----           |                         |                 |             |                 | -----  10. Verifies
2. Store           --5. Requested message-->|  Relay network  |--9.(b, p)-->|     xClient     |      |      proof
   messages                                 |                 |             |                 |<-----  
                                            +-----------------+             +-----------------+
                                             |      ^                             |     ^
                                             |      |                             |     |
                                              ------                               -----
                                          Message process                     11. Updates header
                                                 |                                and remove quest
                                                 |
                                                 v
6. Sort messages according to the ledger index and deserialize all messages to get the hashed data field and decoded public key field. 
7. Convert decoded message format to the format that is compatible to the supported proving system.
8. If using ZKP, invoke the proof generator to generate ZK proof for processed validation messages. 

```



### 1.2 Changes to the XRPL and Rippled
This proposal does not introduce any change to XRPL infrastructure. 
If instantiating an XRPL full node with a validator, a tracking server, or a Clio server, 
then some new Remote procedure call (RPC) methods should be provided by these servers. 
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
A full node in xClient ecosystem is the node that has a complete copy of transaction history and the current state, either for the whole XRPL ledgers or some most recent ledgers. 
Anyone can join the XRPL network and act as a full node to provide services for others. 
We propose two main basic functionalities of a full node: 

1. Storing the whole XRPL transaction history and keeping the current ledger state up to date. We refer the node with full ledger history since the genesis ledger as the archive node, and the node with partial history (e.g., most recent 1024 ledgers) as the regular full node. 

2. Storing the validation messages of recent ledgers (e.g., for most recent 1024 ledgers). 

3. Responding to queries from other full nodes and relayers for information about ledger headers and transactions.

In order to respond to queries about the information of ledger headers and inclusion proof, 
a full node must support following RPC methods.  

### 3.1 `getledgerheaders`
Returns the header of a specific XRPL ledger header or a set of ledger headers, identified by ledger hashes or heights. 

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


## 4. xClient
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

### 4.1 Security 

This design adopts the security definition from the peer-reviewed [zkBridge](https://arxiv.org/pdf/2210.00264) paper 
and revises the definition accordingly to fit XRPL requirements. 
Let $\lambda$ be the security parameter. 
Recall that any ledger header commits to a chain of ledgers back to the Genesis ledger. 
Let $r$ be the round counter that indicates the time of ledger header updates in xClient, 
we denote as $C^P_r$ the chain to which a party $P$'s most recent validated ledger commits to in a given round $r$. Similarly, we denote $C^{X_i}_r$ the chain to which 
a XRPL light client $X_i$’s most recently accepted header commits to. 
xClient should satisfy the following properties: 

* **Safety.** A (light) client protocol is safe 
if for any xClient $i$ the following holds with overwhelming probability in $\lambda$: 
  1. Self-consistency: For any round $`r_1 \leq r_2`$, $`C^{X_i}_{r_1} \preceq C^{X_i}_{r_2}`$.
  2. View-consistency: For any round $r$ and any honest full node $P$, 
  it holds that $`C^{X_i}_{r} \preceq C^{P}_{r}`$, or $`C^{X_i}_{r} \preceq C^{P}_{r}`$.
* **Liveness.** A (light) client protocol is $u$-live if for any transaction
$tx$ an honest consensus/full protocol node receives in any round $r$, 
$tx$ will be reflected in $C^{X_i}_{r+u}$ with overwhelming probability in $\lambda$.
* **Efficiency.** A (light) client protocol is efficient, if it has the following two properties: 
  1. It has a constant size of proof for safety and liveness.
  2. It has a constant time to verify the proof of a new ledger header for updates. 
  
<!-- let $View_i = [lh_1,lh_2,...,lh_i]$ where $lh_i$ indicates the ledger header at height $i$, 
xClient should satisfy the following properties: 
- **Succinctness**: For each state update, xClient should take constant 𝑂(1) time to synchronize the state.
- **Liveness**: If an honest full node receives some transaction $tx$ at some round $𝑟$, 
then $tx$ must be included into the XRPL eventually. 
xClient will eventually include a ledger header $lh_k$ such that the corresponding ledger includes the transaction $tx$. 
- **Consistency**: For any round of $i$ and $j$, it must be satisfied that either $View_𝑖$ is a prefix of $View_j$, or vice versa.  -->

### 4.2 Header Proof
In xClient model, either full nodes or relayers should provide validity proof for new ledger headers in order to achieve safety and liveness properties. 
In this design, we propose **validation message-based proof** for updating ledger headers on xClient. 
**In particular, the design assumes an xClient instantiation maintains its own UNL 
which should also have $`90\%`$ overlap with the consensus node in the XRPL network (assumption 3)**.
When a new ledger is created, relayers obtain the validation messages from XRPL network which 
are signed by XRPL validators, generate a proof for the validation messages, and relay the messages along with the proof to xClient. 
**The design assumes the use of signature schemes that are secure under existential unforgeability against chosen message attacks (assumption 4)**. 
In XRPL, validation messages are signed by ECDSA with Secp256k1 curve. 
After receiving the required validation messages, xClient verifies proof against the validation messages and the UNL it maintains. 
If the proof indicates that majority of the signatures are valid, then xClient accepts the new ledger header. 

In XRPL consensus mechanism, to avoid forking, 
the UNLs of validators should have a high degree of overlap. 
Similarly, in this design, an xClient instantiation also uses the recommended 35 validators in default to configure its UNL. 
If more than 80% validators in xClient's UNL sign a new ledger header in validation messages, 
xClient consider the new header as valid and updates its internal state accordingly. 

We also proposes to leverage ZK-SNARK technique to achieve the efficiency property. 
This specification does not specify which ZK-SNARK proving system to be utilized in xClient design. 
A developer may choose a preferred proving system. 
However, the selected system should generate a constant size proof and has a constant verification time (e.g., Groth16). 
**This specification also assumes that the setup phase in the selected system should 
correctly generate the structured reference string and 
satisfy the completeness, soundness, and zero-knowledge (if required) properties (assumption 5)**
\xnote{add details of proving system?}

An example of a validation message from XRPL:
```
{
  "cookie": "10245873374866816142",
  "data": "2280000001260040E981292F26CC633A8E30A78CAA91BC8E513F79ABC7F620F35A82AF0D2CB5CC0397E0208B4CC22DB3DCDCBD7CB49E6D2C705017000000000000000000000000000000000000000000000000000000000000000050193E48C53A3CAE64E296B14D286E23E18D5611889F74AFA927C52C94FC16002D907321028C9C1DE3789DA22316D789E31099D10F0FE5977DAFD45459B1311FFB65F46FC976473045022100AC7604E7C9747FE6BF3CAE8505662EAFB2564FE6999D1182F417565C3D224A950220416E7E9A8E849AF36A7130E22B7F0846BD0CA621198097884C75CBADB54B64B9",
  "flags": 2147483649,
  "full": true,
  "ledger_hash": "3F79ABC7F620F35A82AF0D2CB5CC0397E0208B4CC22DB3DCDCBD7CB49E6D2C70",
  "ledger_index": "4254081",
  "master_key": "nHUeUNSn3zce2xQZWNghQvd9WRH6FWEnCBKYVJu2vAizMxnXegfJ",
  "signature": "3045022100AC7604E7C9747FE6BF3CAE8505662EAFB2564FE6999D1182F417565C3D224A950220416E7E9A8E849AF36A7130E22B7F0846BD0CA621198097884C75CBADB54B64B9",
  "signing_time": 791071843,
  "type": "validationReceived",
  "validated_hash": "3E48C53A3CAE64E296B14D286E23E18D5611889F74AFA927C52C94FC16002D90",
  "validation_public_key": "n9KcRZYHLU9rhGVwB9e4wEMYsxXvUfgFxtmX25pc1QPNgweqzQf5"
}
```
<!--**![diagram](https://github.com/ZhangxiangHu-Ripple/XRPL-specs/blob/main/XLS-zkLightclient/xclient_relayer.png)**-->

### 4.3 Update mode
An xClient should support one of the following modes to update its internal state. 

#### 4.3.1 Active update
In the active update, xClient is always synchronized with XRPL mainnet, regardless of whether xClient is invoked or not. 
A relayer that subscribes to XRPL validations streams receives validation messages from XRPL network. 
Then the relayer generates a validity proof and sends the proof along processed new ledger message to xClient. 
xClient actively listens to the new ledger message and 
always updates its internal state when the message is arrived. 

#### 4.3.2 Passive update
In the passive update, xClient is synchronized with XRPL mainnet only if it receives a service quest from a user, 
regardless of whether xClient is invoked or not. 
When xClient receives a user's quest, it stores the quest in `questQueue` and performs one of the follows:
* The xClient sends a RPC request to its connected relayers for synchronization with XRPL mainnet. 
* Relayers monitor the `questQueue` and sends requested ledger message to fulfill the quest. 

#### 4.3.3 Hybrid update 
In the hybrid update, xClient is synchronized with XRPL mainnet when it is invoked. 
When xClient is invoked, it sends a message to its connected relayers to notify the start of synchronization. 
Then the relayers act as in the active update mode to send proofs and new ledgers to xClient for synchronization. 

### 4.4 Containers
This section specifies the containers in xClient.

#### 4.4.1 `metaData`
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

#### 4.4.2 `consensusData`
A `consensusData` container includes the consensus information that is used to verify a new XRPL ledger header. 

|Field Name|JSON Type|Description|
|-------|---------|---------|
|`timeStamp`|`Time`| The timestamp of the last verified ledger.
|`consensusType`|`String`| XRPL consensus mechanism.
|`validators`|`Array of Strings`| The UNL of xClient.
|`validatorKeys`|`Array of Strings`| The public keys of accepted validators if `proofType` has the value of 0.

#### 4.4.3 `questQueue`
A `questQueue` container includes the quests that xClient requires: `update` to update the latest ledger header and `inclusionProof` to inquire the inclusion proof of a transaction. 
A user sends a task to xClient and xClient stores the task in `questQueue`. 
A relayer picks a task from `questQueue` and provide the corresponding information to complete the task. 

|Field Name|JSON Type|Description|
|-------|---------|---------|
|`quests`|`Array of String`| A queue that stores all unfinished tasks. 

#### 4.4.4 `quest`
A `quest` container describes a task that the xClient needs to perform. 
A task can be generated by xClient or a user that connects to xClient. 
This proposal defines two types of quest:
1. Update quest. It is generated by xClient or a user for the update of specified XRPL ledger header.
2. Proof quest. It is generated by user and submitted to xClient for the proof of a transaction. 

|Field Name|JSON Type|Description|
|-------|---------|---------|
|`address`|`String`| A unique ID of xClient instantiation. For example, an Ethereum address. 
|`type`|`String`| Update quest or Proof quest. 
|`Quest`|`Object`| Quest details. 
|`fee`|`Number`| The reward for relayers to complete the task. 

### 4.5 Methods
This section defines the publicly exposed methods in xClient. 

#### 4.5.1 `ledgerUpdate`
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
#### 4.5.1 `getLedger`
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

#### 4.5.2 `sendQuest`
`sendQuest` takes the input of a quest from a user and insert the quest into `questQueue`

Example:
```
def sendQuest(Quest: quest) {
  # The quest can be ledger header update, inclusion proof quest, and other defined quests. 
  quest_insert(quest, questQueue)
}
```

#### 4.5.3 `pendQuest`
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

#### 4.5.4 `removeQuest`
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

## 5. Relay Network
The xClient ecosystem requires a relay service to forward required information from 
XRPL mainnet or full nodes to xClient for header update and transaction verification. 

1. A relay node continuously monitor XRPL new ledger headers. 
When a new ledger header is created, the node obtains the new ledger header information by either 
  a) monitoring the XRPL mainnet (by subscribing Validations Stream) or 
  b) querying full nodes with `getledgerproof` method. 
Then the node generates the header proof with **zero-knowledge proof** technique and forwards the new ledger header along with the header proof to an xClient instance. 

2. A relay node monitors xClient request queue and retrieve requests from xClient. 

3. A relay node submits requests to full nodes via full node RPC and obtains the requested information. 

4. A relay node forwards the requested information to xClient.

4. (Optional) In case of XRPL consensus mechanism change, a relayer submits the evidence of the change to xClient. 

In this proposal, a relayer is only responsible for establishing communications between XRPL mainnet/full nodes and 
does not perform any verification of transactions or ledger headers, thus do not need to be trusted to behave honestly. 
However, to simplify the case of DoS, in the relay network, 
**we assume there is at least one honest node who connects to at least one honest full node. 
The honest relayers relay headers, transactions, and the validity proofs between full nodes and xClients (Assumption 2)**.

A relayer in xClient model is an abstract entity. 
Anyone can join the relay network to provides relay service between XRPL and xClient. 
For example, a node can act as both a full node and a relayer. 
In practice, a potential instantiation of a relayer is the witness server, as described in 
[XLS-38](https://github.com/XRPLF/XRPL-Standards/tree/d61629cf2a5ad81153c7439f74f48d90dc96e741/XLS-0038-cross-chain-bridge).

The workflow of a relayer is as follows:
1. A relayer fetches a quest from xClient's `questQueue`. 
2. The relayer processes the quest, submits it to full nodes via RPC or subscribes to XRPL validation messages. 
3. The relayer obtains the response from the full node or XRPL mainnet. 
4. The relayer processes the obtained response and runs a prover to generate a corresponding proof according to the quest.
5. The relayer forwards the response along with the proof to xClient. 

# Appendix

## Appendix A: FAQ

### A.1: How to instantiate an XRPL light client?

An xClient can be instantiated by a wallet, a dApp, and an on-chain smart contract.

### A.2: Why do we need an on-chain xClient?

An on-chain xClient can be used in other applications such as zkBridge and cross-chain lending.

### A.3: What is the incentive to run a full node, a relayer, or an xClient?

A full node can charge a subscription fee for relayers to use its RPC service. A relayer can earn rewards by completing xClient quests. An xClient can charge service fee for providing service to users and applications. 

### A.4: Why do we need to use ZK to verify the validity of a ledger header. 

Because verifying multiple signatures and state change is expensive, especially for an on-chain light client. 

## Appendix B: Pseudocode Example

### B.1: Define SNARK relation 
* Function $`\Pi_{UNL}(\Phi, w)`$
  * if $`\nexists S  \subseteq UNL: |S| \geq 0.8 \cdot |UNL| \wedge \forall pk_i \in S: \exists \sigma_i \in w: \sum.Vrfy(\Phi, pk_i, \sigma_j)`$ then
    * return False
  * if $`\neg valid_header(\phi)`$ then
    * return  False
  * return true

![SNARK relation](URL_or_path_to_image)













