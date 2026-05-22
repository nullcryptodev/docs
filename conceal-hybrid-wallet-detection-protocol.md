# Conceal Wallet Detection Protocol

## Abstract

The Conceal Wallet Detection Protocol enables lightweight, efficient, and privacy-preserving wallet synchronization through
a dedicated on-chain filter database. A single compact data structure stored alongside each block allows wallets to identify
their outputs by downloading approximately 40 megabytes of filter data rather than multiple gigabytes of full block data. The
protocol requires a network fork to activate but provides a permanent, universal, and backward-compatible solution for fast
wallet sync across the entire chain history.

---

## 1. Problem Statement

Cryptocurrency wallets face a fundamental challenge: determining which blockchain transactions belong to them without downloading
and scanning every block. In CryptoNote-based networks, outputs are stealth-addressed. The output key is derived from the
recipient's public keys and cannot be linked to the recipient without possessing the corresponding private view key. This provides
strong privacy but means wallets must either download all block data and cryptographically test every output against their keys,
or query a server that has precomputed the mapping from outputs to public keys. The first approach is slow but private. The second
is fast but reveals wallet activity to the server.

Neither approach is ideal. Full scanning takes 30 to 60 minutes for a 2-million-block chain and requires the full blockchain to be
available locally. Server-side queries require the server to maintain expensive indexes and compromise user privacy.

We propose a protocol that places compact filter data directly on-chain, activated through a network fork. This allows wallets to
identify their outputs with minimal data transfer while maintaining cryptographic verification of ownership.

---

## 2. Design Principles

**On-Chain Permanence.** The filter data is committed to the blockchain at the protocol level. Every full node stores and validates
it. Wallets can trust that the data is complete and correct without trusting any individual server.

**Minimal Storage Overhead.** The filter data must be small enough that it does not meaningfully increase the storage burden on full
nodes. We target approximately 8 bytes per output, or 40 megabytes total for a chain with 5 million outputs.

**Cryptographic Verification.** Wallets must independently verify that any output identified by the filter actually belongs to them.
The filter is a hint, not a proof. False positives are acceptable at a low rate. False negatives are not acceptable under any
circumstances.

**Universal Coverage.** The filter data covers every block from genesis to the current tip. There is no distinction between old
and new transactions. The wallet can sync against any block in the chain's history with equal efficiency.

**No Trusted Server Required.** Once a wallet has the block headers and filter data, it can determine which blocks contain its
outputs without any further trust in the serving node. The filter data is self-authenticating through the block header chain.

---

## 3. The Filter Database

### 3.1 Structure

Each block in the chain has an associated filter record. The filter record contains, for every `KeyOutput` in the block, the
following fields:

| Field | Size | Description |
|-------|------|-------------|
| `view_tag` | 1 byte | First-pass filter derived from the transaction public key and view secret key |
| `nullifier` | 4 bytes | Second-pass confirmatory filter derived from the output key and block height |
| `output_index` | 1 byte | Position of this output within its transaction |
| `tx_index` | 2 bytes | Position of this transaction within the block |

Total per output: 8 bytes. For a block with 10 outputs, the filter record is 80 bytes plus a 4-byte header specifying the output
count.

The complete filter database for a 2-million-block chain with 5 million outputs is approximately 40 megabytes.

### 3.2 View Tag (1 Byte)

The view tag is the first byte of a hash combining the transaction public key, the recipient's view secret key, and the output index:

```
view_tag = first_byte( cn_fast_hash( txPubKey || viewSecretKey || outputIndex ) )
```

A wallet that possesses the view secret key can compute the view tag for any transaction output and compare it against the stored
value. If the tags differ, the output is definitely not owned by that wallet. If they match, the output may be owned by that wallet,
with a 1 in 256 probability of a false positive.

The view tag eliminates 99.6% of outputs from further consideration at the cost of 1 byte per output.

### 3.3 Nullifier (4 Bytes)

The nullifier is the first four bytes of a hash combining the output key and the block height:

```
nullifier = first_4_bytes( cn_fast_hash( outputKey || blockHeight ) )
```

The output key is itself derived from the transaction public key, the key derivation, and the recipient's spend public key. All of
these values can be computed locally by the wallet once it has the block header.

The nullifier provides a confirmatory second check. With 4 bytes, the probability of a false match is 1 in 4,294,967,296. Across 5
million outputs, the expected number of false positives is essentially zero.

### 3.4 Combined Filter Logic

The two filters work in sequence. The view tag provides a cheap first pass that eliminates the vast majority of outputs. The
nullifier provides a precise second pass that eliminates the remaining false positives from the view tag stage.

The output index and transaction index allow the wallet to directly locate the matching output within the block without scanning
every transaction. Once the wallet downloads a block identified by the filter, it can jump directly to the specified transaction
and output index to extract the amount, verify the output key, and check spent status.

---

## 4. Fork Activation

The filter database is activated through a scheduled network fork at a predetermined block height. After the fork height, the filter
record is a mandatory part of each block. Nodes that do not include a valid filter record produce invalid blocks that are rejected
by the network.

### 4.1 Pre-Fork Blocks

Blocks created before the fork height do not have filter records in their original form. To provide universal coverage, nodes build
the filter records for pre-fork blocks retroactively during initial blockchain synchronization. This is a one-time computation
performed as the node processes each historical block. The resulting filter records are stored in the same database and served
to wallets identically to post-fork records.

From the wallet's perspective, there is no distinction between pre-fork and post-fork blocks. All blocks have filter records
available.

### 4.2 Post-Fork Blocks

After the fork height, the miner or block producer is responsible for computing the filter record and including it in the block.
The filter record is committed to by the block hash and validated by all nodes. A block with an incorrect or missing filter record
is rejected.

The computation of the filter record is deterministic and uses data already available during transaction construction. The sender
knows the transaction public key (which they generated), the output key (which they derived), and the view tag (which they can
compute if the recipient's view public key is known). In practice, the filter record can be finalized by any full node after the
block is assembled, since all inputs to the computation are present in the block data.

### 4.3 Validation Rules

Each node validates the filter record for every block after the fork height according to the following rules:

- The number of entries in the filter record must match the number of `KeyOutput` entries in the block.
- Each view tag and nullifier must be correctly computed from the corresponding output data.
- The output indices and transaction indices must correctly reference the positions within the block.

A block that fails filter record validation is rejected. This ensures that wallets can trust the filter data without independently
verifying it against the full block.

---

## 5. Wallet Synchronization

### 5.1 Bootstrap (First Sync)

A wallet performing its first synchronization follows these steps.

**Step 1: Download block headers**

The wallet downloads all block headers from genesis to the current tip. Each header is approximately 56 bytes and includes the block
version, timestamp, previous block hash, nonce, and the coinbase transaction public key. For a 2-million-block chain, this is
approximately 112 megabytes.

The headers establish the chain of trust. The wallet verifies the proof-of-work chain and confirms it has the correct blockchain.

**Step 2: Download filter database**

The wallet requests the complete filter database. This is a binary blob containing the filter records for every block from genesis
to the current tip. For a chain with 5 million outputs, this is approximately 40 megabytes.

The filter database can be served through a simple HTTP endpoint. The wallet downloads it in a single request.

**Step 3: Local filter checking**

Using the block headers and filter database, the wallet processes each block locally:

1. For each block, extract the transaction public keys. There is one per transaction in the block, available from the block data or
2. filter record.
3. For each transaction public key, generate the key derivation.
4. For each filter entry in the block's filter record:
   - Pass 1 (View Tag Check): Compute the view tag. If it does not match the stored view tag, skip this entry.
   - Pass 2 (Nullifier Check): If the view tags matched, compute the output key and then the nullifier. If the nullifier matches
   - the stored value, this entry corresponds to an output we own. Record the block height, transaction index, and output index.

The two-pass structure means the wallet only performs the more expensive key derivation and nullifier computation for the 0.39% of
entries that pass the view tag check.

**Step 4: Request matching blocks**

The wallet now has a precise list of block heights that contain its outputs. It requests only these blocks from the network. For a
typical wallet, 100 to 200 blocks will match. Each block is a few kilobytes. Total additional download: approximately 0.5 megabytes.

**Step 5: Extract outputs**

For each downloaded block, the wallet uses the recorded transaction index and output index to jump directly to the output in
question. It verifies the output key, extracts the amount, and records the output. For full wallets with a spend key, it derives the
key image to enable spent detection.

**Bootstrap Summary**

| Component | Size |
|-----------|------|
| Block headers | 112 MB |
| Filter database | 40 MB |
| Matching full blocks | 0.5 MB |
| **Total** | **152.5 MB** |

Expected total time: 20 to 55 seconds on a typical broadband connection.

### 5.2 Incremental Sync

After the initial bootstrap, the wallet stays synchronized by periodically downloading new block headers and filter records for
blocks since its last scan. Each incremental sync processes at most a few hundred new blocks and downloads a few hundred kilobytes
of data. Processing time is under one second.

The wallet checks the daemon's current chain height, either by polling every 30 seconds or by subscribing to height change
notifications. When new blocks are detected, it downloads only the filter records for the new height range. It runs the same
two-pass filter check locally. If any new blocks match, it downloads those specific blocks and extracts the outputs.

### 5.3 View-Only Wallets

View-only wallets possess the view secret key and spend public key, but not the spend secret key. They can perform all steps in
the synchronization flow except deriving key images, which requires the spend secret key.

To determine which of their outputs have been spent, view-only wallets query the daemon for key image data specific to their owned
outputs. Since the wallet has already narrowed its outputs to a few hundred through the filter process, this query is small and
efficient.

---

## 6. Privacy Analysis

The filter protocol improves privacy compared to server-side output delivery systems.

In a server-side query system, the wallet asks the server for its outputs directly. The server learns exactly which outputs the
wallet owns, including amounts and transaction associations.

In the filter protocol, the wallet downloads two public datasets. Block headers and filter records are identical for all wallets.
The wallet then requests specific full blocks. The server learns which blocks the wallet requested, but not which outputs within
those blocks belong to the wallet. A block containing 10 outputs where only 1 belongs to the wallet reveals only that the wallet
had some activity in that block. It does not reveal which output, what amount, or whether the output has been spent.

The 1-byte view tag does not reveal which view key produced it. The tag is a hash incorporating the view secret key. Without
knowing the view secret, an observer cannot determine which wallet the tag corresponds to. Since the tag is only 1 byte, 256
different wallets could produce the same tag for different outputs, providing a form of k-anonymity.

The 4-byte nullifier is a hash of the output key and block height. It reveals nothing about the view key or spend key. It merely
confirms the existence of a specific output key at a given block height. An observer who has not already identified the output
key cannot link the nullifier to any wallet.

The wallet requests approximately 100 to 200 blocks out of 2 million. This is a sparse access pattern that varies between wallets
based on their specific output distribution. This is comparable to the privacy model of Bitcoin SPV wallets.

---

## 7. Security Considerations

**False negatives are impossible.** The view tag and nullifier are both deterministic functions of keys the wallet possesses. If
an output belongs to the wallet, the wallet will always compute matching tags and nullifiers. There is no scenario where a wallet
misses its own outputs due to filter failure.

**False positives are cryptographically bounded.** The view tag has a 1/256 false positive rate by design. The nullifier has a
1/4,294,967,296 false positive rate. The combined false positive rate is approximately 1 in 1.1 trillion. In practice, the
nullifier eliminates essentially all view tag false positives.

**A malicious node cannot cause the wallet to accept fake outputs.** The wallet independently verifies every output by checking
the key derivation and comparing output keys. Even if a node sends incorrect filter data or fabricated blocks, the wallet's local
cryptographic verification will reject outputs that do not match its keys.

**A malicious node cannot hide outputs from the wallet.** The filter records are committed to by the block hash and validated by
the network. A node cannot serve fake filter records for post-fork blocks without the records failing validation. For pre-fork
blocks, the wallet can cross-check filter data from multiple nodes.

**Filter record validation prevents poisoning.** Because the filter record is validated by every node as part of block verification,
a miner cannot include incorrect filter data that would cause wallets to miss their outputs or download unnecessary blocks.

---

## 8. Implementation Plan

### Phase 1: Protocol Specification

Define the exact binary format of the filter record. Specify the fork height at which filter records become mandatory. Document
the validation rules that nodes must enforce.

### Phase 2: Filter Record Computation

Implement the filter record computation in the block assembly and validation code. Add the filter record to the block structure.
Implement validation that rejects blocks with incorrect or missing filter records after the fork height.

### Phase 3: Retroactive Build

Implement the retroactive filter record builder that processes pre-fork blocks during initial sync and populates the filter
database. This is a one-time scan similar to the existing index rebuild process.

### Phase 4: Storage and Serving

Add the filter record to the MDBX storage layer. Implement the binary RPC endpoint for serving filter data to wallets. Implement
the incremental endpoint for new blocks.

### Phase 5: Wallet Integration

Update the wallet client to use the filter protocol. Implement the two-pass filter checking logic. Implement direct output
extraction using the transaction index and output index from the filter record.

### Phase 6: Testing and Activation

Test the protocol on testnet. Verify filter record validation, wallet sync performance, and backward compatibility with pre-fork
clients. Activate the fork on mainnet at the scheduled height.

---

## 9. References

### View Tags

The view tag concept was introduced by UkoeHB in the Monero Research Lab and implemented by j-berman in the Fluorine Fermi network upgrade.

- UkoeHB. "Reduce scan times with 1-byte-per-output view tag." Monero Research Lab Issue #73. [https://github.com/monero-project/research-lab/issues/73](https://github.com/monero-project/research-lab/issues/73)
- j-berman. "Add view tags to outputs to reduce wallet scanning time." Pull Request #8061, monero-project/monero. [https://github.com/monero-project/monero/pull/8061](https://github.com/monero-project/monero/pull/8061)
- "Implement view tags to decrease wallet sync times in Monero." Monero Bounty, 23.624 XMR. Completed by j-berman. [https://bounties.monero.social/](https://bounties.monero.social/)

### Bloom Filters and Compact Client-Side Filtering

Probabilistic data structures for efficient blockchain querying.

- Bitcoin Improvement Proposal 37. "Connection Bloom Filtering." [https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki)
- Bitcoin Improvement Proposal 157. "Client-Side Block Filtering." [https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki)
- Bitcoin Improvement Proposal 158. "Compact Block Filters for Light Clients." [https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki)

### CryptoNote Protocol

The foundation of the transaction protocol used by Conceal Network.

- van Saberhagen, Nicolas. "CryptoNote v2.0." October 2013. [https://cryptonote.org/whitepaper.pdf](https://cryptonote.org/whitepaper.pdf)

### MDBX Database

The embedded database engine used for blockchain storage.

- ReOpenLDAP Project. "libmdbx." [https://libmdbx.dqdkfa.ru/](https://libmdbx.dqdkfa.ru/)

### Privacy-Preserving Blockchain Queries

Academic work on private information retrieval for blockchain data.

- Bunz, Benedikt and Kiffer, Lucianna and Lu, Shengyun and Wang, Xiao. "Riptide: Fast, Private, and Accountable Cryptocurrency Light Clients." 2023. [https://eprint.iacr.org/2023/131](https://eprint.iacr.org/2023/131)

### Stealth Addressing and Key Derivation

The cryptographic primitives underlying output ownership verification.

- Courtois, Nicolas and Mercer, Rebekah. "Stealth Address and Key Management Techniques in Blockchain Systems." Proceedings of the 3rd International Conference on Information Systems Security and Privacy, 2017.
