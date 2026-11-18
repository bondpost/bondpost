# Developer guide

## What you are building

You are building a protocol first, a reference implementation second, and products third. Keep them separate in your head and in your repository.

| Layer | What it is | Where it lives | Who else can build it |
| --- | --- | --- | --- |
| Protocol | Rules on paper: the bond script, the proof event, the verification steps, the weight formula | The [spec repository](https://github.com/bondpost/spec), later a draft NIP | Anyone, in any language, without your code |
| Reference implementation | The library and CLI that follow those rules and publish test vectors | This repository | Others can fork or reimplement |
| Products | The verifier service, relay plugin and board | Same repository or separate ones | Anyone, on top of the library |

The protocol is the long-term asset. A product can be replaced, but an event format that two independent projects read is hard to displace. It is also what a grant funder can justify: open rules that others build on.

**Before you start**

- Comfortable Rust, including traits, error handling with `Result`, and writing tests. If Rust is new, work through The Rust Programming Language book first.
- A running Bitcoin Core node in regtest mode on your own machine.
- Willingness to read specifications slowly. Each milestone below names what to read before you write code.

**One rule above all others**

Never write your own elliptic curve, hashing, signing or script-number code. Use the `secp256k1` and `bitcoin` crates for all of it. Your code decides what to compute. The libraries do the computing.

## Reading path

Read in this order. Each row says what to take from the text and which milestone needs it, so you never read ahead of the code you are writing.

| # | Read | Take from it | Needed for |
| --- | --- | --- | --- |
| 1 | [BIP39](https://en.bitcoin.it/wiki/BIP_0039) | Words to seed: 12 to 24 words, the checksum, the optional passphrase | M4 |
| 2 | [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), [BIP43](https://bips.dev/43/), [BIP44](https://bips.dev/44/) | Child key derivation, hardened versus normal, what the purpose and coin type levels mean | M1 |
| 3 | [BIP340, tagged hashes section](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) | Why hashes are tagged and how | M1 |
| 4 | [BIP341, design section](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) | The pay-to-contract idea: Q = P + hash(P, m)G. Bondpost uses the same construction with ECDSA keys | M1 |
| 5 | [BIP141](https://bips.dev/141/) and the [SegWit wallet developer guide](https://bitcoincore.org/en/segwit_wallet_dev/) | P2WSH: single SHA256 of the witness script, what goes in the witness, size limits | M1, M2 |
| 6 | [BIP173](https://bips.dev/173/) | Bech32 addresses and the network prefix | M1 |
| 7 | [BIP65](https://bips.dev/65/) and [Bitcoin transaction timelocks](https://thecharlatan.github.io/Locktime/) | The five ways CLTV fails, and how nLockTime and nSequence interact | M2 |
| 8 | [Miniscript, BIP379](https://bips.dev/379/) | The translation table, enough to see why the bond script is `and_v(v:pk(K),after(L))` | M1 |
| 9 | [BIP143](https://bips.dev/143/) | What a SegWit signature commits to: amount, sequence, locktime, script | M2 |
| 10 | [BIP46](https://bips.dev/46/), [Belcher's design](https://gist.github.com/chris-belcher/18ea0e6acdb885a2bfbdee43dcd6b5af) and [bond mathematics](https://gist.github.com/chris-belcher/87ebbcbb639686057a389acb9ab3e25b) | How the closest prior system derives, proves and values a bond | M1, M5, decisions D-1 and D-2 |
| 11 | [defiads announcement thread](https://mirror.b10c.me/lists/bitcoindev/f3d9000a-beef-a10c-5aa4-30bbda966c24@gmail.com/T) and [README](https://github.com/defiads/defiads) | The original design and the use cases its authors had in mind | Context |
| 12 | [NIP-01](https://nips.4rs.nl/nips/1) | Event structure, id and signature, addressable events, the `a` and `d` tags, filters | M3 |
| 13 | [NIP-40](https://nips.4rs.nl/nips/40), [NIP-46](https://nips.4rs.nl/nips/46) | Expiration tags, remote signing | M6 |
| 14 | [gettxout](https://bitcoincore.org/en/doc/27.0.0/rpc/blockchain/gettxout) | The one RPC verification depends on | M4 |
| 15 | [NIP-69](https://nips.4rs.nl/nips/69), [NIP-99](https://nips.4rs.nl/nips/99) | The offer formats you will rank | M8 |
| 16 | [Esplora API](https://raw.githubusercontent.com/Blockstream/esplora/master/API.md), [strfry plugins](https://github.com/hoytech/strfry/blob/beta/docs/plugins.md) | Light verification and relay policy | M8 |
| 17 | [OrangeCheck specification](https://github.com/orangecheck/oc-attest-protocol) | The neighbouring protocol: its message format, lifecycle and conformance vectors are a model for how to publish yours | M9 |

Two books cover rows 1 to 9 at tutorial pace if the specifications feel dense: Mastering Bitcoin and Programming Bitcoin. For Rust APIs, keep the documentation for the `bitcoin`, `secp256k1`, [`nostr-sdk`](https://docs.rs/crate/nostr-sdk/latest) and [`bdk_wallet`](https://docs.rs/bdk_wallet) crates open while coding.

## Milestones

Eleven milestones, each small enough to finish and test before the next. Story ids refer to [user stories](user-stories.md).

**M0 Repository and tooling**

- Build: a Cargo workspace with `bondpost-core` and `bondpost-cli` crates, continuous integration running tests and `clippy`, an MIT or Apache-2.0 licence file, a README that states the project is experimental.
- Done when: an empty test passes in CI on every push.

**M1 Derivation and script** (stories F-2, A-2)

- Read: rows 2 to 6 and 8 of the reading path.
- Build: `derive_bond`. Inputs are a base public key, a Nostr public key, a locktime and a network. Outputs are the bond key, witness script, scriptPubKey and address.
- Done when: a vectors file with at least ten fixed cases passes (committed to the spec repository's [`test-vectors/`](https://github.com/bondpost/spec/tree/main/test-vectors)), the same inputs give the same address on two machines, and changing any input changes the address.
- Cross-check: build the descriptor `wsh(and_v(v:pk(K),after(L)))` with the `miniscript` crate and assert it produces byte-identical script.

**M2 Reclaim transaction and dry run** (story A-5)

- Read: rows 7 and 9.
- Build: `build_reclaim`, signing with the tweaked private key. Then the dry run: build a synthetic funding output with the real script, sign the reclaim against it, and run the full witness through a consensus script verifier.
- Done when: a property test over random keys and locktimes never produces a bond the dry run cannot spend.

**M3 Proof event and verification** (stories C-2, C-4)

- Read: row 12.
- Build: `build_proof`, `parse_proof`, and `verify` against a `ChainSource` test double that returns canned outputs.
- Done when: each of the nine failure codes is triggered by its own crafted input, and no test touches the network.

**M4 Bitcoin Core source, wallet and CLI** (stories F-1, A-1, A-3, A-4, A-6, C-1, D-1)

- Read: rows 1 and 14.
- Build: a `ChainSource` backed by Core RPC, seed creation and restore, per-network directories, then `init`, `bond new`, `bond fund`, `bond list` and `verify`.
- Done when: you fund a bond on regtest and a second config directory verifies it.

**M5 Weight, reclaim, recovery** (stories C-3, D-2 to D-5)

- Read: row 10.
- Build: `weight`, `bond reclaim`, `bond export` and `import`, recovery from proof events.
- Done when: the [regtest walk-through](test-plan.md#regtest-walk-through) runs unattended in CI, with the proof passed between the two config directories as a file, including the early-spend rejection at tip L minus 1 and acceptance at tip L.

**M6 Publishing** (stories B-1 to B-3)

- Read: row 13.
- Build: `bond publish` with a local key, with `--unsigned` and `--signed`, and relay responses reported per relay. Add NIP-46 last.
- Done when: a proof published to a local relay is fetched back by a different process and verifies, and the full walk-through, publishing included, runs unattended in CI. This is the regtest gate.

**M7 Signet and light verification** (stories E-3, E-4, E-5, D-6)

- Read: the Esplora half of row 16.
- Build: the Esplora `ChainSource` with multi-server agreement, reorganisation handling, the verifier HTTP service, seed-only scan recovery.
- Done when: the service has run on signet for 30 days without a wrong answer.

**M8 Relay plugin and board** (stories E-1, E-2, C-5, C-6, B-4)

- Read: row 15 and the strfry half of row 16.
- Build: the strfry write-policy plugin, then a static web board that ranks NIP-99 listings by weight.
- Done when: ten people outside the project complete create, publish and reclaim on signet without your help.

**M9 Specification and outreach** (story F-5)

- Read: row 17.
- Build (in the spec repository): the draft NIP, a conformance vectors file others can run, and a short comparison with OrangeCheck. Settle decisions D-1 to D-6.
- Done when: the draft is posted for comment and at least two projects that display offers have replied. This is the signet gate.

**M10 Mainnet beta** (stories F-3, F-4)

- Build: the amount cap, the risk notice, reproducible builds with signed checksums.
- Done when: an outside reviewer has read the M1 and M2 code, their findings are fixed and published, and a 10,000-sat rehearsal bond has been verified by a second person and reclaimed.

M1 to M3 hold every line that can lose money and need no node or relay. Expect them to take longer than they look, and do not move on until the property tests are boring.

## Core logic sketch

This is the whole fund-handling core in outline. It is pseudocode, has not been compiled or run, and is here to show the shape. Your test vectors, not this sketch, decide what is correct.

```
# 1. Derivation (M1)
t_bytes = tagged_hash("Bondpost/bond/v1", base_pk[33 bytes] || nostr_pk[32 bytes])
t       = scalar_from_big_endian(t_bytes)     # fail if 0 or >= curve order
K       = base_pk + t*G                       # library tweak-add, fail on infinity
k       = (base_sk + t) mod n                 # library tweak-add on the secret key
assert public_key_of(k) == K                  # always check, in code and in tests

# 2. Script and address (M1)
witness_script = PUSH(K as 33 bytes) OP_CHECKSIGVERIFY PUSH_INT(L) OP_CHECKLOCKTIMEVERIFY
script_pubkey  = OP_0 PUSH(sha256(witness_script))        # single SHA256
address        = bech32(prefix_for(network), version 0, sha256(witness_script))

# 3. Reclaim (M2)
tx.version   = 2
tx.lock_time = L                               # a block height
tx.inputs    = [ { outpoint, sequence = 0xfffffffd, script_sig = empty } ]
tx.outputs   = [ { destination_script, amount - fee } ]
digest       = bip143_sighash(tx, input 0, script_code = witness_script, amount, SIGHASH_ALL)
signature    = ecdsa_sign(k, digest) || 0x01
tx.inputs[0].witness = [ signature, witness_script ]

# 4. Verification (M3)
check event id and signature
read d, v, network, base, locktime from tags; fail on anything missing or malformed
recompute K, witness_script, script_pubkey from (base, event.pubkey, locktime)
utxo = chain.utxo(outpoint)                    # fail if none
fail unless utxo.script_pubkey == script_pubkey
fail unless utxo.confirmations >= min_conf
fail unless locktime > chain.tip_height()
weight = utxo.amount * min(locktime - tip_height, 52560)
```

In Rust the relevant pieces are the tweak-add methods on `secp256k1::PublicKey` and `SecretKey`, the script builder and `SighashCache` in the `bitcoin` crate, and the tagged-hash helpers in `bitcoin::hashes`. Method names change between releases, so read the documentation for the versions you pin.

**Mistakes this design is exposed to**

| Mistake | Effect | Guard |
| --- | --- | --- |
| Encoding L by hand | A non-minimal number makes the output unspendable under standard rules | Use the library's integer push. Test L = 1, 16, 17, 127, 128, 32767, 32768 |
| Double SHA256 for the script hash | Coins sent to an address nobody can spend | Vectors. BIP141 uses a single SHA256 here |
| Txid byte order | Verification looks up the wrong outpoint | Use the library's `Txid` type end to end, never raw bytes |
| Sequence left at 0xffffffff | CLTV fails and reclaim is rejected | Dry run in M2 |
| `nLockTime` below L | Same | Dry run in M2 |
| Amount handled as a float | Wrong weight, wrong fee | Integer sats everywhere, parse Core's BTC value with the library's `Amount` |
| Trusting the `amount` tag | Fake weight | Story C-1 test |
| Using the wrong network's coin type or prefix | Test proofs that look like mainnet | Per-network directories, vectors per network |
| Losing the base key index | Bond cannot be rebuilt | Bond file, plus recovery from proof events |
| Signing with the base key instead of the tweaked key | Reclaim fails after funding | The assert in step 1 and the dry run |

## What was verified on 17 November 2026

The five settled standards all say what the specification assumes. The purpose number and the parameters are choices, so they were checked for conflicts and arithmetic, not for truth.

**Settled standards**

| Standard | What the specification relies on | Result |
| --- | --- | --- |
| [BIP32, BIP43, BIP44](https://bips.dev/44/) | Hardened levels for purpose, coin type and account. Coin type 0 for mainnet, 1 for every test network | Confirmed |
| [BIP39](https://en.bitcoin.it/wiki/BIP_0039) | 12 to 24 words from 128 to 256 bits of entropy, optional passphrase in the salt | Confirmed |
| [BIP141](https://bips.dev/141/) | The witness is the input stack followed by the witness script, whose single SHA256 must equal the 32-byte program | Confirmed |
| [BIP143](https://en.bitcoin.it/wiki/BIP_0143) | The signature commits to the script, the amount spent, `nSequence` and `nLockTime` | Confirmed |
| [BIP173](https://bips.dev/173/) | `bc` for mainnet, `tb` for testnet | Confirmed, with one note below |

Notes from the reading:

- `bcrt` for regtest is a Bitcoin Core convention, not part of BIP173. Signet shares `tb` with testnet.
- Relay policy caps a P2WSH witness script at 3,600 bytes and stack items at 80 bytes. The bond script is about 41 bytes and its one signature at most 73, so both limits are far away.
- Policy also treats a P2WSH output under 330 sats as dust. The 10,000-sat minimum is well above it.

**The purpose number 8421**

It collides with nothing, and it is also not sanctioned by anything.

- BIP43 asks each scheme to get a BIP number and use that number as its purpose. There is no BIP 8421.
- BIP43 reserves 10001 to 19999 for SLIPs. 8421 is outside that range.
- So the number is free today but only by convention. If the specification ever becomes a BIP or a SLIP, switch to the assigned number in a new protocol version, since changing the path changes every address.
- The alternative is BIP46's approach of a new branch under purpose 84. That risks ordinary wallets tripping over the branch, which is why the specification keeps a separate purpose.

**Design parameters**

None of these is a fact that can be right or wrong. Each is listed with its basis so you can change it knowingly.

| Parameter | Value | Basis | Checked |
| --- | --- | --- | --- |
| Maximum term and weight cap | 52,560 blocks | 144 blocks a day at the 10-minute target, times 365 | Arithmetic correct |
| Locktime rounding | Multiples of 144 | One day. Keeps the recovery scan to 20 indexes times about 730 heights | Arithmetic correct |
| Minimum bond | 10,000 sats | Arbitrary floor against dust-sized spam | About 30 times the dust limit |
| `min_conf` on mainnet | 6 | Long-standing convention for settled payments | Convention, not a rule |
| Mainnet beta cap | 1,000,000 sats | Bounds the loss from an unknown bug | Arbitrary, decision D-4 |
| Verify time target | 200 ms | A guess at what feels instant | Measure in M4 |
| Weight formula | sats times blocks remaining | Simplest form of cost of capital | See below |

On the weight formula: JoinMarket values a bond as the square of the coins times a compound-interest term in the lock time. For short terms that interest term is close to linear in time, so the two agree on time and differ on amount, where JoinMarket squares. That is decision D-1, and Belcher's mathematics paper in the reading path is the argument for squaring.

## Long term and grants

Apply after the regtest gate at the earliest, and preferably once signet is public, because the application asks for things you can only show by then. Only OpenSats was checked here. Other funders exist and were not researched.

[OpenSats](https://opensats.org/apply) funds free and open-source work in Bitcoin and Nostr, reviews proposals on a rolling basis, and names three criteria as crucial: good for Bitcoin, free and open-source, and transparency and education. Closed-source work is not funded.

**What the [application template](https://opensats.org/static/opensats-grant-application-template.md) asks for, and what to prepare**

| The application asks for | Prepare this | Ready by |
| --- | --- | --- |
| Repository and open-source licence | Public repository from M0, licence file in place | M0 |
| Prior contributions | A visible history: commits, issues or reviews on Rust Bitcoin, Nostr or related projects | Start now |
| At least two written references from people in the Bitcoin or open-source community | Relationships. Post progress publicly, ask for review of the draft NIP, contribute a fix upstream | M9 |
| Project description and potential impact | Two paragraphs: the spam problem, and what a consensus-enforced bond adds that existing tools lack | M5 |
| Screenshots or videos | A recording of the regtest walk-through, later the signet board | M5, M8 |
| Timeline and milestones | M6 to M10 from this guide, with their done-when lines | M5 |
| Duration of 3, 6, 9 or 12 months and a time commitment | A realistic choice. OpenSats suggests considering a shorter 3 or 6 month grant | Application |
| Budget in USD with a breakdown | Your time, plus the outside review in M10 | Application |
| Prior and additional funding | An honest list, even if empty | Application |
| A two-minute video | You explaining the project plainly | Application |

Evaluation does not begin until two references have arrived, so that row is the one to start earliest.

**What makes the project last**

- A specification others can implement without reading your code, with conformance vectors.
- One integration you did not write. A relay operator or a client reading bond weight is worth more than any feature.
- Public, boring maintenance: tagged releases, a changelog, issues answered.
- A clear statement of what the project is not. Say plainly how it differs from OrangeCheck and from JoinMarket fidelity bonds, and credit defiads, Tamas Blummer and ZmnSCPxj for the origin.
