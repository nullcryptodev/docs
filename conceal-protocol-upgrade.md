# Conceal Network Protocol Upgrade: Self-Describing Outputs, Encrypted Memos, and On-Chain DNS

## Overview

This document describes a single hard fork that introduces four new output types to the Conceal Network. Together they deliver
faster wallet scanning, private payment memos, and a human-readable domain name system with built-in privacy tiers. Every output
becomes a self-contained package carrying all the information a wallet needs to identify ownership, decrypt associated data, and
resolve domain names, all without external indexes or fragile state tracking.

The fork activates at a fixed block height to be determined. All nodes and wallets must upgrade before that height. Old output
types remain valid for pre-fork blocks. Post-fork blocks use the new output types exclusively for the features described here.

---

## The New Output Types

### 0x04 — Standard Payment Output

This replaces the current `KeyOutput` for post-fork blocks. It carries everything a wallet needs to recognize the output in a
single pass.

| Field | Size | Description |
|-------|------|-------------|
| `view_tag` | 1 byte | First byte of `H("view_tag" \|\| txPubKey \|\| varint(output_index))`. Eliminates 99.6 percent of false outputs before any expensive ECDH derivation. |
| `key_index` | 2 bytes | The derivation index for this output. The wallet passes this directly to `derive_public_key`. No more global key counter. No more tracking different index schemes for KeyOutput and MultisigOutput. |
| `memo_size` | 1 byte | Length of the encrypted memo field in bytes, from 0 to 255. A value of zero means no memo is present. |
| `encrypted_memo` | variable | A blob encrypted with the shared secret between sender and recipient. Contents are opaque to observers. The wallet decrypts this after confirming ownership. Can carry payment IDs, subaddress indices, amounts, refund addresses, or any future data. |
| `key` | 32 bytes | The one-time public key, same as the current `KeyOutput`. |

Total size for a standard output with a typical 8 byte memo is 44 bytes. The current output is 32 bytes. The 12 byte increase is
a 37 percent increase per output, which translates to roughly 10 to 15 percent larger blocks overall.

### 0x05 — Multisig Payment Output

This extends the multisig output with the same self-describing fields. It also introduces a flags byte that distinguishes between time-locked deposits (for staking) and flexible authorized deposits (for the sidechain bridge and other off-chain protocols like HTLCs).

| Field | Size | Description |
|-------|------|-------------|
| `view_tag` | 1 byte | Same as 0x04. |
| `key_index` | 2 bytes | Base derivation index for the multisig group. Each key within the group uses `key_index + position`. |
| `num_keys` | 1 byte | Number of keys in the multisig group. |
| `flags` | 1 byte | Bit field controlling how this output can be spent. `0x01` means time-locked. `0x02` means authorized. Both can be combined as `0x03`. All other bits are reserved for future use. |
| `term` | 4 bytes | Block count for fixed lockup. Only enforced by consensus when `flags & 0x01` is set. For flexible deposits without a time lock, this field is zero and ignored. |
| `memo_size` | 1 byte | Same as 0x04. |
| `encrypted_memo` | variable | Same as 0x04. |
| `keys` | 32 bytes each | The multisig public keys. |

The `flags` field enables three modes of operation.

A time-locked deposit sets `flags = 0x01` with a nonzero term. The output cannot be spent until the block height reaches the
output's creation height plus the term. This is used for staking deposits where funds must remain locked for a fixed period to
earn rewards. Consensus enforces the time lock at block validation time.

A flexible authorized deposit sets `flags = 0x02` with a term of zero. The output can be spent at any time, but only with a valid
signature from an authorized key. The authorized key is one of the keys in the multisig group, designated by convention (for
example, the first key). This is used by the sidechain bridge, which manages its HTLC logic off-chain. The bridge verifies the
HTLC conditions externally and signs the withdrawal when they are satisfied. Consensus only checks that the signature is valid.
It does not know or care about the HTLC logic. This keeps the HTLC entirely off-chain while using the same output type as staking
deposits.

A combined deposit sets `flags = 0x03` with a nonzero term. The output has both a minimum lock time and an authorization
requirement. It cannot be spent before the term expires, and even after expiry it requires the authorized signature. This is
useful for bridged assets that also earn staking rewards during the lock period.

If neither flag is set, the output behaves exactly like the current multisig output with no additional restrictions.

### 0x06 — Domain Registration

This output registers a human-readable domain name on the blockchain. It is not a spendable output. The registration fee is paid
separately via a standard 0x04 output included in the same transaction, sent to the team's address. This keeps the domain output
itself clean and non-spendable.

| Field | Size | Description |
|-------|------|-------------|
| `view_tag` | 1 byte | Always zero. This output is not spendable and is identified by its type byte. |
| `key_index` | 2 bytes | Always zero. |
| `domain_len` | 1 byte | Length of the domain name string. |
| `domain` | variable | The domain name as ASCII text, for example `alice.conceal`. |
| `tier` | 1 byte | Privacy tier. 1 means fully private, only view key holders can resolve. 2 means the view key is shared with specific parties like a regulator or accountant. 3 means the view key is public and anyone can resolve the domain. |
| `domain_pub` | 32 bytes | A signing key controlled by the domain owner. Used to prove ownership for updates, renewals, or deletion. |
| `domain_view_pub` | 32 bytes | A view key for domain resolution. Holders of the corresponding secret view key can decrypt the address field to discover where the domain points. |
| `encrypted_addr_size` | 1 byte | Always 64 bytes, containing the encrypted spend public key and subaddress. |
| `encrypted_addr` | 64 bytes | The actual address data encrypted with the domain view key. Only someone with the domain view secret key can decrypt this to learn the destination address. |
| `metadata_len` | 1 byte | Length of optional metadata. |
| `metadata` | variable | Free-form field for compliance certifications, business information, social links, or any future use. |

The daemon maintains a simple index mapping domain names to the block height and output index of their registration. This index
is built during initial sync and updated with each new block. It is tiny because the number of registered domains is expected to
be small relative to the blockchain.

### 0x07 — Domain Deletion

This output removes a domain from the active index. It proves ownership via a signature from the domain signing key.

| Field | Size | Description |
|-------|------|-------------|
| `view_tag` | 1 byte | Always zero. |
| `key_index` | 2 bytes | Always zero. |
| `domain_len` | 1 byte | Length of the domain name. |
| `domain` | variable | The domain name to delete. |
| `sig` | 64 bytes | A signature over the domain name and a deletion marker, verifiable with the `domain_pub` from the original
registration. |

---

## Wallet Scanning After the Fork

The scanning process becomes dramatically simpler. For each output in a post-fork block, the wallet performs these steps.

First, it reads the type byte. If the type is 0x04 or 0x05, it proceeds. If the type is 0x06 or 0x07, it skips the output entirely
because these are domain records, not spendable outputs.

Second, it reads the view tag. It computes the expected view tag from `H("view_tag" || txPubKey || varint(output_index))` and
compares the first byte. If the bytes do not match, the output is not ours. The wallet moves to the next output with zero ECDH
cost. This eliminates 255 out of every 256 wrong outputs.

Third, it reads the key index directly from the output. It calls `derive_public_key(derivation, key_index, spendPub, derivedKey)`
and checks if the derived key matches the output key. If it matches, the output is ours. The wallet has now confirmed ownership.

Fourth, if the memo size is greater than zero, the wallet decrypts the encrypted memo using the shared secret. It can then extract
whatever data the memo contains, such as a payment ID, subaddress index, amount, or refund address. The decryption key is derived
as `H("conceal_memo" || derivation)` and the nonce is the output's global index, making each memo's encryption unique.

For 0x05 outputs, the wallet also reads the flags field to understand the spending conditions. If the time-locked flag is set,
the wallet knows the output cannot be spent until a specific height and displays this in the user interface. If the authorized
flag is set, the wallet knows that an external signature is required and can coordinate with the bridge or other off-chain protocol
as needed.

This entire process requires no external filter database, no global key counter, no special coinbase handling, and no fragile index
tracking. Every output carries everything the wallet needs. Outputs can be checked in any order, making scanning fully
parallelizable across threads.

For pre-fork blocks, the existing scanning code with the filter database continues to work unchanged.

---

## Domain Resolution

Domain resolution works similarly. When a user wants to send to `alice.conceal`, the wallet asks the daemon for the domain record.
The daemon looks up the domain in its index and returns the full 0x06 registration output, plus a Merkle proof that the output
exists in the block. This allows light clients to verify the domain without trusting the daemon.

The wallet extracts the domain view public key from the registration. If the domain is Tier 3, the view key is already public and
the wallet can immediately decrypt the address. If the domain is Tier 1 or Tier 2, the sender needs the domain view secret key,
which must be obtained from the domain owner through an out-of-band channel. This is similar to sharing a payment address today,
except the domain name is the public identifier and the view key controls who can resolve it.

Once the wallet has the view key, it decrypts the `encrypted_addr` field to recover the spend public key and subaddress. It then
constructs a normal 0x04 payment to that address. The payment itself is indistinguishable from any other transaction. Only someone
with the domain view key knows that a particular transaction was sent to a particular domain.

The three privacy tiers serve different use cases. Tier 1 domains are completely private. Only people who the owner explicitly
shares the view key with can resolve the name. This is the default and works like sharing an address today, except the name is
reusable and the key sharing is a one-time event. Tier 2 domains are for businesses and exchanges that need to share their records
with a regulator or auditor. The view key is given to the specific party who needs it. The public cannot resolve the name, but the
regulator can audit all transactions. Tier 3 domains are fully public. Anyone can resolve the name and verify on-chain that the
domain points to a specific address. This is ideal for charities, open source projects, and businesses that want maximum
transparency.

---

## Coinbase Uniformity

Currently coinbase transactions do not include a transaction public key. This forces wallets to special-case the coinbase when
scanning blocks. After the fork, coinbase transactions will include a randomly generated transaction public key just like every
other transaction. The wallet scanning loop no longer needs any special handling for the coinbase. Every transaction in every
block follows the same format.

---

## Filter Database Legacy

The filter database that was built for pre-fork blocks will no longer be extended after the fork. New blocks will not have filter
records because the view tags are now on-chain. The daemon can optionally prune old filter records, keeping only the most recent
window of blocks (for example, the last 100,000). This reduces daemon storage from hundreds of megabytes to roughly 20 megabytes
for the filter database. Eventually, as the chain grows past the pruning window, the filter database will disappear entirely.

---

## Encrypted Memo Encryption Details

The encrypted memo uses ChaCha20 with the key derived from `H("conceal_memo" || derivation)` and the nonce set to the output's
global index. This ensures that even if two outputs share the same derivation, their memo encryptions produce different
ciphertexts. The memo is encrypted by the sender and decrypted by the recipient using the same shared secret.

Optionally, a Poly1305 authentication tag can be appended to the memo for integrity verification. This adds 16 bytes per memo.
Without the tag, the recipient relies on the output key ownership check for integrity. If the output key matches, the memo is
authentic because only the true sender could have derived the shared secret that encrypted it. This is a design decision to be
finalized before implementation.

---

## Domain Registration Fees

Registering a domain requires a payment of 100 CCX to the team's address. This payment is made via a standard 0x04 output included
in the same transaction as the 0x06 registration output. The payment is hidden among the ring signatures of the transaction,
preserving the privacy of the registrant.

This creates a sustainable funding stream for ongoing development. It is entirely optional. Users who do not want a domain are
unaffected. Users who want domains pay a one-time fee and own the name permanently. If squatting becomes a problem, the fee can
be adjusted or a renewal requirement can be added in a future fork.

---

## Daemon Indexes

The daemon maintains two simple indexes after the fork.

The domain index maps domain names to the block height and output index of their registration. This index is built during initial
sync by scanning all post-fork blocks and is updated incrementally with each new block. It is tiny, roughly 100 bytes per registered
domain, and can be stored in memory or in MDBX. Lookups are O(1) hash map operations. The index is purely derived from on-chain
data and can be rebuilt from scratch at any time.

The filter database index for pre-fork blocks continues to exist but is no longer appended to after the fork. Old records can be
pruned to a configurable window. This is a maintenance task that runs periodically.

No other indexes are required. The domain index and the prunable filter database are the only daemon-side data structures needed.

---

## Upgrade Mechanics

The hard fork activates at a specific block height. Before that height, the network operates exactly as it does today. At the fork
height, nodes begin enforcing the new output types and rejecting old output types in new blocks. Wallets begin using the new
scanning logic for blocks at and after the fork height while continuing to use the existing scanning logic with the filter database
for earlier blocks.

All nodes must upgrade before the fork height. Exchanges, pools, and services must upgrade their nodes and wallet infrastructure.
Users must upgrade their wallets. This is a one-time coordination event. The upgrade can be signaled through the existing checkpoint
system, social channels, and a new client version that refuses to connect to pre-fork peers after the fork height.

---

## Summary of Benefits

Wallet scanning becomes dramatically faster and simpler. The view tag eliminates 99.6 percent of unnecessary ECDH derivations. The
key index field eliminates an entire category of bugs related to global key counter tracking. The encrypted memo field provides
private, extensible metadata without any on-chain leakage. The domain system creates human-readable names with flexible privacy
controls and generates sustainable funding for development.

The multisig output gains a flags field that supports both fixed-term staking deposits and flexible authorized deposits for the
sidechain bridge and other off-chain protocols. The bridge can manage its HTLC logic entirely off-chain while using the same output
type as staking deposits. The consensus layer enforces the time lock when requested and stays out of the way when not needed.

The codebase shrinks because entire subsystems like the filter record builder, filter RPC handler, and key index tracking logic can be removed for post-fork blocks. Future wallet implementations are easier to write and test because outputs are fully self-describing.

The cost is a 10 to 15 percent increase in block size, a one-time hard fork coordination effort, and the implementation work to
support the new output types. These costs are well understood and manageable. The benefits are permanent and compound over time as
more users join the network.
