![](https://user-images.githubusercontent.com/1248214/57789522-600fcc00-7739-11e9-86d9-73d7032f40fc.png)

[![Build and Test](https://github.com/KILTprotocol/kilt-collator/workflows/Build%20and%20Test/badge.svg)](https://github.com/KILTprotocol/kilt-collator/actions)

# KILT kilt-collator <!-- omit in toc -->

The KILT parachain collators use Parity Substrate as the underlying blockchain
technology stack extended with our DID, CType, Attestation and hierarchical Trust Modules.

-   [1. How to use](#1-how-to-use) - [1.1. How to use TL;DR](#11-how-to-use-tldr) - [1.2. How to use longer](#12-how-to-use-longer) - [1.3. Relay Chain: Polkadot](#13-relay-chain-polkadot) - [1.3.1. Start Alice's Node](#131-start-alices-node) - [1.3.2. Start Bob's Node](#132-start-bobs-node) - [1.4. KILT Collator](#14-kilt-collator) - [1.4.1. Generate Genesis State](#141-generate-genesis-state) - [1.4.2. Obtain WASM Validation Function](#142-obtain-wasm-validation-function) - [1.4.3. Start Collator Node](#143-start-collator-node) - [1.4.4. Register parachain in Apps](#144-register-parachain-in-apps)
-   [2. Node Modules functionalities](#2-node-modules-functionalities) - [2.1. DID Module](#21-did-module) - [2.1.1. Add](#211-add) - [2.1.2. CRUD](#212-crud) - [2.2. CTYPE Module](#22-ctype-module) - [2.3. Attestation Module](#23-attestation-module) - [2.3.1. Add](#231-add) - [2.3.2. Revoke](#232-revoke) - [2.3.3. Lookup](#233-lookup) - [2.4. Hierarchy of Trust Module](#24-hierarchy-of-trust-module) - [2.4.1. Create root](#241-create-root) - [2.4.2. Add delegation](#242-add-delegation) - [2.4.3. Revoke](#243-revoke)

**Substrate Documentation**

-   [Cumulus Parachain Workshop](https://substrate.dev/cumulus-workshop/#/)
-   [JSON-RPC](https://polkadot.js.org/api/substrate/rpc.html)
-   [Substrate Reference Rust Docs](https://substrate.dev/rustdocs)

## 1. How to use

### 1.1. How to use TL;DR

TODO

### 1.2. How to use longer

1. Spin up Polkadot validators (number of parachains + 1)
2. Spin up Collator(s)

I recommend checking out the [cumulus-workshop](https://substrate.dev/cumulus-workshop/#/3-parachains/1-launch) and following most of the steps described there, mainly 3. All following code is basically copied from there to have a one page overview for all required code. Please check out the workshop for explanations. The workshop targets

-   Polkadot @ `fd4b176`
-   Substrate Parachain Template @ `2710c42`
-   Polkadot JS Apps @ `46734ee`

### 1.3. Relay Chain: Polkadot

You need to run 2 validators in the relay chain, e.g. `#parachains + 1`. See [here](https://substrate.dev/cumulus-workshop/#/2-relay-chain/2-launch). It should at least work until commit

```bash
cargo build --release
```

#### 1.3.1. Start Alice's Node

```bash
target/release/polkadot \
  --chain rococo_local.json \
  --tmp \
  --ws-port 9944 \
  --port 30333 \
  --alice
```

#### 1.3.2. Start Bob's Node

```bash
target/release/polkadot \
  --chain rococo_local.json \
  --tmp \
  --ws-port 9955 \
  --port 30334 \
  --bob \
  --bootnodes /ip4/<Alice IP>/tcp/30333/p2p/<Alice Peer ID>
```

### 1.4. KILT Collator

Follow the instructions of [section 3](https://substrate.dev/cumulus-workshop/#/3-parachains/1-launch).

#### 1.4.1. Generate Genesis State

```
target/release/kilt-collator export-genesis-state --parachain-id 200 > para-200-genesis
```

#### 1.4.2. Obtain WASM Validation Function

```
target/release/kilt-collator export-genesis-wasm > para-200-wasm
```

#### 1.4.3. Start Collator Node

This assumes your KILT collator repo is a sibling of your Polkadot repo used for the relay chain validators due to using the same precompiled local rococo chainspec.

```
target/release/kilt-collator \
  --tmp \
  --ws-port 9977 \
  --port 30336 \
  --parachain-id 200 \
  --validator \
  -- \
  --chain ../polkadot/rococo_local.json \
  --bootnodes /ip4/<Alice IP>/tcp/30333/p2p/<Alice Peer ID> \
  --bootnodes <Other Relay Chain Node(s)
```

#### 1.4.4. Register parachain in Apps

1. Head to [Polkadot Apps](https://polkadot.js.org/apps/#/?rpc=ws://localhost:9944) and connect to the default local WS address.
2. Sudo > Registrar > registerPara
    1. `id: 200`
    2. `info: ParaInfo` should be `Always`
    3. `code`: Add `para-200-wasm` file located in the root of your cumulus dir
    4. `initial_head_data`: Add `para-200-genesis` from previous step
3. Collator should start producing parachain blocks (aka collating) once the registration is successful. The collator should start producing log messages like the following:

```
2020-06-23 07:28:24 Starting parachain attestation session on top of parent 0xd94fe34cd9708145308d7a2c1f1c0f1105997c5307258970249fd66e27d571bc. Local parachain duty is None
2020-06-23 07:28:24 🙌 Starting consensus session on top of parent 0x74be3a0a708a44fe5f2e16e1829ff3254d6870580297627c54af99a3ecdcfddf
2020-06-23 07:28:24 🎁 Prepared block for proposing at 1 [hash: 0x8ffba7e6194a3a960f9b4b7d0ce1d419d655f5f258f609eb910ddccb78b3d45c; parent_hash: 0x74be…fddf; extrinsics (3): [0xfe5c…5be2, 0x6d07…cb5a, 0x2259…cec2]]
2020-06-23 07:28:24 ✨ [Parachain] Imported #1 (0x8ffb…d45c)
```

## 2. Node Modules functionalities

The KILT blockchain is the heart and soul behind KILT protocol. It provides the immutable
transaction ledger for the various processes in the network.

`Building on the Parity Substrate Blockchain Framework`

During our first whiteboard phase, we were thinking about developing the KILT protocol on
Ethereum smart-contracts, but we realised that we would have less freedom of setting
transaction costs, while incurring a high level of overhead. Instead, we started our
development on [Parity Substrate](https://www.parity.io/substrate/), a general blockchain framework, and built up the KILT blockchain from scratch based on its module library.

Building our blockchain on Parity Substrate has multiple advantages. Substrate has a very
good fundamental [architecture](https://docs.substrate.dev/docs/architecture-of-a-runtime) and [codebase](https://github.com/paritytech/substrate) created by blockchain experts. Substrate
framework is developed in Rust, a memory efficient and fast compiled system programming
language, which provides a secure environment with virtually no runtime errors. Moreover, the
node runtime is also compiled to WebAssembly, so older version native nodes can always run
the latest version node runtime in a WebAssembly virtual machine to bypass the problem of a
blockchain fork. Importantly, there is a vibrant developer community and rich [documentation](https://docs.substrate.dev/).

Our implementation is based on the [substrate-node-template](https://github.com/rstormsf/substrate-node-template) library (skeleton template for
quickly building a substrate based blockchain), which is linked to the main Substrate
codebase.

**Remote Procedure Calls**

The Ethereum ecosystem highly leverages [JSON-RPC](https://www.jsonrpc.org/specification) where one can efficiently call methods and parameters directly on the blockchain node. Based on good experiences, developers
decided to use it in Substrate as well. The [Polkadot API](https://polkadot.js.org/api/) helps with communicating with the JSON-RPC endpoint, and the clients and services never have to talk directly with the endpoint.

**Blocktime**

The blocktime is currently set to 5 seconds, but this setting is subject to change based on
further research. We will consider what is affected by this parameter, and in the long term it
will be fine-tuned to a setting that provides the best performance and user experience for the
participants of the KILT network.

**Extrinsics and Block Storage**

In Substrate, the blockchain transactions are abstracted away and are generalised as
[extrinsics](https://docs.substrate.dev/docs/extrinsics) in the system. They are called extrinsics since they can represent any piece of information that is regarded as input from “the outside world” (i.e. from users of the network) to the blockchain logic. The blockchain transactions in KILT are implemented through these
general extrinsics, that are signed by the originator of the transaction. We use this framework
to write the KILT Protocol specific data entries on the Substrate based KILT blockchain: [DID],
[CTYPEhash], [Attestation] and [Delegation]. The processing of each of these entry types is
handled by our custom Substrate runtime node modules.

Under the current consensus algorithm, authority validator nodes (whose addresses are listed
in the genesis block) can create new blocks. These nodes [validate](https://docs.substrate.dev/docs/transaction-lifecycle-in-substrate) incoming transactions, put
them into the pool, and include them in a new block. While creating the block, the node
executes the transactions and stores the resulting state changes in its local storage. Note that
the size of the entry depends on the number of arguments the transaction, (i.e., the respective
extrinsic method) has. The size of the block is hence dynamic and will depend on the number
and type of transactions included in the new block. The valid new blocks are propagated
through the network and other nodes execute these blocks to update their local state (storage).

**Authoring & Consensus Algorithm**

We use [Aura](https://wiki.parity.io/Aura) as the authoring algorithm, since we are still in a permissioned blockchain mode.
We will probably move to another algorithm in the future (e.g. [BABE](https://w3f-research.readthedocs.io/en/latest/polkadot/BABE.html)).

For consensus we use [GRANDPA](https://github.com/paritytech/substrate#2-description).

At a later stage, we most likely will change to a different consensus algorithm that will incorporate additional features (e.g. proving availability of certain services) and we might leverage concepts from BABE+GRANDPA while designing this new consensus mechanism.

**Polkadot Integration**

As a further great advantage, by basing ourselves on Substrate we can easily connect to the
Polkadot ecosystem. This could provide security for the KILT network by leveraging the global
consensus in the Polkadot network. We are planning to integrate KILT into the [Polkadot](https://polkadot.network/)
network. It is fairly straightforward to achieve that by simply including specific Substrate
modules into the KILT Validator Node implementation. The exact details of this integration is
subject to future agreements between Polkadot and KILT and the technological development
of Polkadot, Substrate and KILT.

**KILT Tokens**

Coin transfer is implemented as a balance-based mechanism in Substrate. In our testnet,
every new identity gets 1000 KILT Tokens from a root entity in the system who is wired into
the genesis block. At a later stage in the testnet (Mash-Net) and the persistent testnet (WashNet), we are proposing to provide KILT Tokens for new developers wanting to join our network
on a simple request-provide based mechanism. Preferably, developers will be able to register
on our website, and we manually transfer KILT tokens to the registered developers.
Importantly, these test tokens will not be usable on our mainnet. After the launch of the mainnet
(Spirit-Net) and the public KILT Token sale, tokens will be available on cryptocurrency
exchanges.

### 2.1. DID Module

The KILT blockchain node runtime defines an DID module exposing:

#### 2.1.1. Add

```rust
add(origin, sign_key: T::PublicSigningKey, box_key: T::PublicBoxKey, doc_ref: Option<Vec<u8>>) -> Result
```

This function takes the following parameters:

-   origin: public [ss58](<https://wiki.parity.io/External-Address-Format-(SS58)>) address of the caller of the method
-   sign_key: the [ed25519](http://ed25519.cr.yp.to/) public signing key of the owner
-   box_key: the [x25519-xsalsa20-poly1305](http://nacl.cr.yp.to/valid.html) public encryption key of the owner
-   doc_ref: Optional u8 byte vector representing the reference (URL) to the DID
    document

The blockchain node verifies the transaction signature corresponding to the owner and
inserts it to the blockchain storage by using a map (done by the substrate framework):

```rust
T::AccountId => (T::PublicSigningKey, T::PublicBoxKey, Option<Vec<u8>>)
```

#### 2.1.2. CRUD

As DID supports CRUD (Create, Read, Update, Delete) operations, a `get(dids)` method
reads a DID for an account address, the add function may also be used to update a DID and
a `remove(origin) -> Result` function that takes the owner as a single parameter removes the DID from the
map, so any later read operation call does not return the data of a removed DID.

### 2.2. CTYPE Module

The KILT blockchain node runtime defines an CTYPE module exposing

```rust
add(origin, hash: T::Hash) -> Result
```

This function takes following parameters:

-   origin: public [ss58](<https://wiki.parity.io/External-Address-Format-(SS58)>) address of the caller of the method
-   hash: CTYPE hash as a [blake2b](https://blake2.net/) string

The blockchain node verifies the transaction signature corresponding to the creator and
inserts it to the blockchain storage by using a map (done by the substrate framework):

```rust
T::Hash => T::AccountId
```

### 2.3. Attestation Module

The KILT blockchain node runtime defines an Attestation module exposing functions to

-   add an attestation (`add`)
-   revoke an attestation (`revoke`)
-   lookup an attestation (`lookup`)
-   lookup attestations for a delegation (used later in Complex Trust Structures)
    on chain.

#### 2.3.1. Add

```rust
add(origin, claim_hash: T::Hash, ctype_hash: T::Hash, delegation_id: Option<T::DelegationNodeId>) -> Result
```

The `add` function takes following parameters:

-   origin: The caller of the method, i.e., public address ([ss58](<https://wiki.parity.io/External-Address-Format-(SS58)>)) of the Attester
-   claim_hash: The Claim hash as [blake2b](https://blake2.net/) string used as the key of the entry
-   ctype_hash: The [blake2b](https://blake2.net/) hash of CTYPE used when creating the Claim
-   delegation_id: Optional reference to a delegation which this attestation is based
    on

The node verifies the transaction signature and insert it to the state, if the provided attester
didn’t already attest the provided claimHash. The attestation is stored by using a map:

```rust
T::Hash => (T::Hash,T::AccountId,Option<T::DelegationNodeId>,bool)
```

Delegated Attestations are stored in an additional map:

```rust
T::DelegationNodeId => Vec<T::Hash>
```

#### 2.3.2. Revoke

```rust
revoke(origin, claim_hash: T::Hash) -> Result
```

The `revoke` function takes the claimHash (which is the key to lookup an attestation) as
argument. After looking up the attestation and checking invoker permissions, the revoked
flag is set to true and the updated attestation is stored on chain.

#### 2.3.3. Lookup

The attestation lookup is performed with the `claimHash`, serving as the key to the
attestation store. The function `get_attestation(claimHash)` is exposed to the outside
clients and services on the blockchain for this purpose.

Similarly, as with the simple lookup, to query all attestations created by a certain delegate,
the runtime defines the function `get_delegated_attestations(DelegationNodeId)`
that is exposed to the outside.

### 2.4. Hierarchy of Trust Module

The KILT blockchain node runtime defines a Delegation module exposing functions to

-   create a root `create_root`
-   add a delegation `add_delegation`
-   revoke a delegation `revoke_delegation`
-   revoke a whole hierarchy `revoke_root`
-   lookup a root `get(root)`
-   lookup a delegation `get(delegation)`
-   lookup children of a delegation `get(children)`
    on chain.

#### 2.4.1. Create root

```rust
create_root(origin, root_id: T::DelegationNodeId, ctype_hash: T::Hash) -> Result
```

The `create_root` function takes the following parameters:

-   origin: The caller of the method, i.e., public address (ss58) of the owner of the
    trust hierarchy
-   root_id: A V4 UUID identifying the trust hierarchy
-   ctype_hash: The blake2b hash of the CTYPE the trust hierarchy is associated with

The node verifies the transaction signature and insert it to the state. The root is stored by using
a map:

```rust
T::DelegationNodeId => (T::Hash,T::AccountId,bool)
```

#### 2.4.2. Add delegation

```rust
add_delegation(origin, delegation_id: T::DelegationNodeId, root_id: T::DelegationNodeId, parent_id: Option<T::DelegationNodeId>, delegate: T::AccountId, permissions: Permissions, delegate_signature: T::Signature) -> Result
```

The `add_delegation` function takes the following parameters:

-   origin: The caller of the method, i.e., public address (ss58) of the delegator
-   delegation_id: A V4 UUID identifying this delegation
-   root_id: A V4 UUID identifying the associated trust hierarchy
-   parent_id: Optional, a V4 UUID identifying the parent delegation this delegation is
    based on
-   CTYPEHash: The blake2b hash of CTYPE used when creating the Claim
-   delegate: The public address (ss58) of the delegate (ID receiving the delegation)
-   permissions: The permission bit set (having 0001 for attesting permission and
    0010 for delegation permission)
-   delegate_signature: ed25519 based signature by the delegate based on the
    delegationId, rootId, parentId and permissions

The node verifies the transaction signature and the delegate signature as well as all other data
to be valid and the delegator to be permitted and then inserts it to the state. The delegation is
stored by using a map:

```rust
T::DelegationNodeId => (T::DelegationNodeId,Option<T::DelegationNodeId>,T::AccountId,Permissions,bool)
```

Additionally, if the delegation has a parent delegation, the information about the children of its
parent is updated in the following map that relates parents to their children:

```rust
T::DelegationNodeId => Vec<T::DelegationNodeId>
```

#### 2.4.3. Revoke

```rust
revoke_root(origin, root_id: T::DelegationNodeId) -> Result
```

and

```rust
revoke_delegation(origin, delegation_id: T::DelegationNodeId) -> Result
```