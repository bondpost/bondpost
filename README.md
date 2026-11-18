# Bondpost

Bitcoin-bonded offers on Nostr. Lock bitcoin to your Nostr identity until a future block height, and let clients and relays rank your offers by that real, consensus-enforced cost. You get every satoshi back when the term ends.

> **Experimental.** Nothing here is ready for mainnet. Use regtest or signet only. A bug in the lock or reclaim code can freeze coins until the timelock expires, or forever.

This repository is the reference implementation: the `bondpost-core` Rust library, the `bondpost` CLI and, later, the verifier service, relay plugin and board. The protocol it implements is specified separately, in the [spec repository](https://github.com/bondpost/spec), so that others can build compatible implementations without reading this code.

## Status

Pre-code. The first milestone is M0 (workspace and CI). See the [developer guide](docs/developer-guide.md#milestones).

## Documentation

See [`docs/`](docs/README.md).

## Licence

Not yet chosen. Proposed: MIT or Apache-2.0 dual licence. See [decision D-5](https://github.com/bondpost/spec/blob/main/decisions/D-5-licence.md) in the spec repository.
