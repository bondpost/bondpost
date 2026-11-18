# Test plan

Every test kind below has a gate, and a release does not ship until its gate's tests pass. The conformance vectors themselves live in the spec repository's [`test-vectors/`](https://github.com/bondpost/spec/tree/main/test-vectors). This repository runs them; it does not define them.

## Regtest walk-through

The first end-to-end target, to automate in CI at milestone M5.

1. Start a node: `bitcoind -regtest -daemon`, create a wallet, then mine 101 blocks with `generatetoaddress` so coins are spendable.
2. Start a local Nostr relay such as strfry.
3. `bondpost init --network regtest`, then `bondpost bond new` with a 10-block term.
4. Send coins to the printed address from the node wallet and mine 1 block.
5. `bondpost bond publish`, then `bondpost verify` from a second config directory: expect success and a weight.
6. Try `bondpost bond reclaim`: expect a refusal, because the bond is not expired.
7. Mine 10 blocks. `bondpost verify` now returns `expired`.
8. `bondpost bond reclaim`, mine 1 block, and confirm the coins are back in the wallet.

Step 6 matters as much as step 8, because it shows that consensus enforces the lock, not the tool. The test suite should also broadcast a correctly signed early spend directly and assert that the node rejects it as non-final.

## Required tests

| Kind | Covers | Gate |
| --- | --- | --- |
| Test vectors | Fixed keys and locktimes giving fixed tweak, K, script, address and event id (from the spec repository) | Regtest |
| Unit | Every verification step, each failure code triggered by a crafted input | Regtest |
| Property | For random keys and locktimes, the dry-run reclaim always validates against the script | Regtest |
| Integration | The walk-through above, automated against a real regtest node and relay | Regtest |
| Negative | Wrong identity, wrong network, altered amount tag, early spend, spent output | Regtest |
| Fuzz | Event parser and tag handling | Signet |
| Reorg | Funding block invalidated with `invalidateblock`, verifier withdraws the bond | Signet |
| Recovery | Wallet deleted, bonds rebuilt from seed by each of the three recovery routes | Signet |
| Soak | Verifier service running 30 days on signet against live relays | Signet |
| Mainnet rehearsal | A 10,000-sat bond with a 144-block term created, verified by a second person, reclaimed | Mainnet beta |
| External review | Derivation, script and reclaim code read by a reviewer experienced in Bitcoin script | Mainnet beta |

## Release gates

| Gate | Passes when |
| --- | --- |
| Regtest exit | Lifecycle tests pass on every run, and there are no open fund-safety bugs |
| Signet exit | 10 testers finish the lifecycle unaided, and 30 days pass with no critical bug |
| Beta exit | One independent verifier exists, and zero sats have been lost or stuck |
