# User stories

## How to read these stories

Thirty-two stories in six epics, each with acceptance criteria you can turn directly into tests.

- Format: "As a \[persona\], I want \[action\], so that \[outcome\]", followed by Given / When / Then criteria.
- Priority: Must, Should or Could within the release named.
- Each story names the functional requirements (FR) it delivers, from [architecture](architecture.md#functional-requirements).

| Persona | Who they are |
| --- | --- |
| Bonder | Posts offers and locks coins to rank them |
| Reader | Browses offers and wants to see fewer throwaway ones |
| Developer | Integrates the library into a client or tool |
| Relay operator | Runs a relay and wants cheap spam control |
| Verifier operator | Runs the HTTP service for light clients |
| Maintainer | You, shipping and supporting releases |

## Epic A: create a bond

Six stories take a bonder from an empty machine to a confirmed, locked output.

**A-1 Set up a wallet** · Must · Regtest · FR-01

As a bonder, I want to create or restore a wallet from a seed phrase, so that my bond keys can always be re-derived.

- Given no wallet exists, when I run `init`, then I am shown a 12 or 24 word seed once and must confirm I saved it.
- Given I have a seed, when I run `init --restore`, then the same base keys are derived as before.
- Given a wallet exists, when I inspect its files, then the seed is not readable without my passphrase.

**A-2 Prepare a bond** · Must · Regtest · FR-02, FR-03

As a bonder, I want to enter my Nostr public key, an amount and a term, so that I get an address that locks coins to my identity.

- Given a valid npub, amount and term, when I run `bond new`, then I see the address, the unlock height and a bond id in state `draft`.
- Given the same seed, index, npub and locktime, when I run it on another machine, then the address is identical.
- Given a different npub, when all else is equal, then the address is different.

**A-3 See what I am committing to** · Must · Regtest · FR-06

As a bonder, I want to see when my coins unlock before I pay, so that I do not lock them longer than I meant.

- Given a draft bond, when the address is about to be shown, then I see the unlock height, an estimated date and the words "cannot be spent before then".
- Given that prompt, when I do not type the confirmation, then no address is shown.

**A-4 Be stopped from unsafe locks** · Must · Regtest · FR-05

As a bonder, I want the tool to refuse extreme terms and amounts, so that a typing mistake cannot freeze my coins for years.

- Given a term above 52,560 blocks, when I run `bond new`, then it is refused with the maximum stated.
- Given the same input plus the explicit override flag, then it proceeds after a second confirmation.
- Given a term that is not a multiple of 144, then it is rounded up and I am told the new unlock height.

**A-5 Know the coins can come back** · Must · Regtest · FR-04, NFR-01

As a bonder, I want the tool to prove it can spend the bond before I fund it, so that I never pay into a script nobody can unlock.

- Given a draft bond, when the address is requested, then a reclaim transaction is built and signed against a synthetic funding output carrying the real script, and its full witness passes a consensus script verifier.
- Given that check fails, then no address is shown and the error is reported.
- Given any combination of flags, then this check still runs.

**A-6 Fund the bond** · Must · Regtest · FR-07

As a bonder, I want to pay the address from any wallet and watch it confirm, so that I know when the bond is live.

- Given a payment to the address, when it enters the mempool, then the bond moves to `funded`.
- Given `min_conf` confirmations, then the bond moves to `active` and shows its outpoint and amount.
- Given two payments to the same address, then each appears as its own bond.

## Epic B: publish and attach the proof

Four stories get the proof event signed, onto relays, and linked from offers.

**B-1 Publish my proof** · Must · Regtest · FR-08, FR-09

As a bonder, I want to publish a proof for an active bond, so that anyone can verify it.

- Given an `active` bond, when I run `bond publish`, then an event of the bond kind with every required tag is sent to my relays.
- Given a `draft` or `funded` bond, then publishing is refused with the reason.
- Given the published event, when it is fetched back from the relay, then it passes verification.

**B-2 Keep my Nostr key out of the tool** · Must · Regtest · FR-08

As a bonder, I want to sign the proof with my own signer, so that I never paste my secret key into new software.

- Given `--unsigned`, when I publish, then the unsigned event JSON is printed and nothing is sent.
- Given a signed event file, when I run `bond publish --signed <file>`, then it is checked against the bond and sent.
- Given a NIP-46 connection string, then the signer is asked to sign and I approve it there. (Should, Signet)

**B-3 Know where my proof landed** · Should · Regtest · FR-09

As a bonder, I want each relay's answer, so that I know my proof is actually available.

- Given three relays, when one rejects the event, then I see which one and its message, and the other two are reported as accepted.
- Given a proof already published, when I add a relay and republish, then the same event id is sent, not a new one.

**B-4 Point an offer at its bond** · Could · Signet

As a developer, I want a helper that returns the `a` tag for a bond, so that my client can attach it to listings and orders.

- Given a verified bond, when I call the helper, then I get the tag in the form the specification defines.
- Given an offer whose author differs from the bond's identity, when weight is computed, then the reference is ignored.

## Epic C: verify and rank

Six stories cover checking a proof, turning it into a weight, and showing that weight to readers honestly.

**C-1 Verify someone's proof** · Must · Regtest · FR-10

As a reader, I want to check a proof event against my own node, so that I do not rely on the bonder's word.

- Given a valid proof for a confirmed, unexpired bond, when I run `verify`, then I see the amount from the chain, the unlock height, the blocks remaining and the weight.
- Given the proof's `amount` tag is altered upward, then the result still shows the on-chain amount.
- Given I have never contacted the bonder, then verification still completes.

**C-2 Understand a failure** · Must · Regtest · FR-10

As a developer, I want exactly one failure code per failed check, so that I can test and report precisely.

- Given a crafted input for each of the nine failure codes, when verified, then that code and no other is returned.
- Given a proof whose `pubkey` was swapped for another identity and re-signed, then the result is `script_mismatch`.
- Given a regtest proof and a signet verifier, then the result is `wrong_network`.

**C-3 See an identity's weight** · Must · Regtest · FR-11

As a reader, I want one number for an identity, so that I can compare two posters.

- Given an identity with two active bonds, when I run `weight`, then the result is the sum of both by the default formula.
- Given one of them expires, when a block passes height L, then its contribution is zero.
- Given a bond of 9,999 sats, then it contributes nothing.

**C-4 Use the library with my own chain access** · Must · Regtest · FR-10, FR-11

As a developer, I want to pass in my own chain source, so that the library fits my client's architecture.

- Given an implementation of the chain interface backed by a test double, when I call `verify`, then no network request is made by the library.
- Given the published test vectors, when my build runs them, then every tweak, script and address matches.

**C-5 Browse offers by weight** · Should · Signet · FR-19

As a reader, I want a board that lists offers with bonded identities first, so that throwaway spam sinks.

- Given listings from five identities with different weights, when the board loads, then they appear in descending weight and unbonded ones come last.
- Given I open a listing's bond detail, then I see the amount, blocks remaining and a link to the output in a block explorer.

**C-6 Not be misled by a bond** · Must · Signet · BR-15

As a reader, I want the board to say what a bond does and does not mean, so that I do not treat it as a guarantee.

- Given any place a weight is displayed, then the label reads as cost committed and never uses the words trusted, verified seller or safe.
- Given the bond detail view, then it states that bonded coins are returned to the bonder and are not compensation for buyers.

## Epic D: reclaim and recover

Six stories make sure every satoshi comes back, including after a lost machine.

**D-1 See my bonds** · Must · Regtest · FR-07, FR-11

As a bonder, I want a list of my bonds with their state, so that I know what is locked and until when.

- Given bonds in several states, when I run `bond list`, then each shows state, amount, unlock height, blocks remaining and current weight.
- Given a bond within 144 blocks of expiry, then it is flagged as expiring soon.

**D-2 Get my coins back** · Must · Regtest · FR-12

As a bonder, I want to reclaim an expired bond, so that my coins return to my ordinary wallet.

- Given an `expired` bond, when I run `bond reclaim`, then a transaction spending it to a fresh wallet address is broadcast and accepted.
- Given that transaction confirms, then the bond is `reclaimed` and the wallet balance rises by the amount less the fee shown beforehand.
- Given no relay and no verifier service is reachable, then reclaim still works with only the chain source.

**D-3 Be told plainly when it is too early** · Must · Regtest · FR-12

As a bonder, I want a clear refusal if I try to reclaim early, so that I do not think the tool is broken.

- Given an `active` bond, when I run `bond reclaim`, then nothing is broadcast and I see the blocks remaining.
- Given the test suite broadcasts a correctly signed early spend directly, then the node rejects it as non-final.

**D-4 Sweep several bonds at once** · Should · Regtest · FR-12

As a bonder, I want one transaction for all my expired bonds, so that I pay one fee.

- Given three expired bonds with different unlock heights, when I run `bond reclaim` with no id, then one transaction spends all three with `nLockTime` equal to the highest.
- Given one of the three is still active, then it is left out and named in the output.

**D-5 Recover on a new machine** · Must · Regtest · FR-13

As a bonder, I want to rebuild my bonds from my seed, so that a dead laptop does not cost me my coins.

- Given my seed and an exported bond file, when I restore and import, then every bond reappears in its correct state.
- Given my seed and my npub but no file, when I run recovery against relays holding my proofs, then every published bond is rebuilt.
- Given a rebuilt expired bond, then reclaim succeeds.

**D-6 Recover without relays** · Should · Signet · FR-20

As a bonder, I want a last-resort scan, so that I can recover even if my proof events are gone.

- Given only my seed and npub, when I run the scan against a full node, then bonds created with default settings are found.
- Given the scan starts, then I am told how many candidate scripts it will check and can cancel it.

## Epic E: relay and verifier operators

Five stories, all for the signet release, let operators use bond weight without writing code.

**E-1 Require a bond to post** · Should · Signet · FR-18

As a relay operator, I want to reject listings from identities below a minimum weight, so that flooding my relay costs real money.

- Given a minimum weight for kind 30402, when an unbonded identity posts a listing, then it is rejected with a message naming the requirement.
- Given an identity above the minimum, then its listing is accepted.
- Given any identity, when it posts a bond proof event, then that event is accepted regardless of weight.

**E-2 Exempt people I know** · Should · Signet · FR-18

As a relay operator, I want an allowlist, so that trusted regulars are not forced to lock coins.

- Given a public key on the allowlist, when it posts a guarded kind with zero weight, then the event is accepted.
- Given I edit the allowlist file, then the change applies without restarting the relay.

**E-3 Serve weights to light clients** · Should · Signet · FR-16

As a verifier operator, I want an HTTP service that answers weight queries, so that phone and web clients need no node.

- Given a known identity, when a client requests its weight, then the response lists each bond and the total, with the tip height used.
- Given the same request repeated, then it is served from cache in under 5 ms.
- Given an unknown identity, then the response is a weight of zero, not an error.

**E-4 Verify without my own node** · Should · Signet · FR-15

As a developer, I want an Esplora chain source with cross-checking, so that one dishonest server cannot fool my users.

- Given two configured servers that agree, when I verify, then the result is returned.
- Given they disagree about an output, then verification fails closed and reports the disagreement.

**E-5 Survive a reorganisation** · Must · Signet · FR-17

As a verifier operator, I want cached results corrected after a reorg, so that I never vouch for a bond that no longer exists.

- Given a bond whose funding block is invalidated on regtest, when the new tip is seen, then the bond is re-checked and withdrawn.
- Given the funding transaction confirms again on the new chain, then the bond returns after `min_conf`.

## Epic F: safety and mainnet readiness

Five stories are the maintainer's own, and the last three are what stand between signet and mainnet.

**F-1 Keep networks apart** · Must · Regtest · FR-14

As a maintainer, I want each network's data in its own directory, so that a test never touches real coins.

- Given wallets on regtest and mainnet, when I run any command with `--network regtest`, then no mainnet file is opened.
- Given no `--network` flag and no config, then the default is regtest, never mainnet.

**F-2 Publish test vectors** · Must · Regtest · NFR-02

As a developer, I want fixed inputs and expected outputs, so that I can write a second implementation and know it agrees.

- Given the vectors file, then it holds at least ten cases covering tweak, bond key, script, address per network and event id.
- Given the reference build, when its test suite runs, then it reads the same file.

**F-3 Cap the beta** · Must · Mainnet beta · FR-21

As a maintainer, I want mainnet bonds limited in size during beta, so that an undiscovered bug has a bounded cost.

- Given mainnet and an amount above the cap, when I run `bond new`, then it is refused and the cap is stated.
- Given first use on mainnet, then the risk notice covering the lock, hot keys, privacy and beta status is shown and must be accepted.

**F-4 Ship builds people can check** · Must · Mainnet beta · FR-22

As a bonder, I want to confirm the binary matches the source, so that I am not trusting the maintainer's laptop.

- Given the tagged source, when two people build it by the documented steps, then the checksums match the published ones.
- Given a release, then its checksums are signed with the maintainer's published key.

**F-5 Get outside eyes on it** · Must · Mainnet beta · BR-13, BR-14

As a maintainer, I want the format and the fund-handling code reviewed by others, so that mainnet does not rest on my judgement alone.

- Given the signet release, then a draft NIP describing the event and verification steps is posted for comment.
- Given the mainnet candidate, then one reviewer experienced in Bitcoin script has read the derivation, script and reclaim code, and their findings are published with the fixes.
- Given the mainnet rehearsal bond, then a second person has verified it and it has been reclaimed.

## Build order

Build the library from the inside out, so that the fund-handling code is finished and tested before any interface exists.

1. Test vectors and derivation: F-2, then the key and script half of A-2. Nothing touches a network yet.
2. Reclaim transaction builder and the dry run: A-5. Prove with property tests that every derived bond can be spent.
3. Proof event build and parse, then verification against a test double: C-2, C-4.
4. Bitcoin Core chain source and network isolation: F-1, C-1.
5. Wallet and CLI lifecycle on regtest: A-1, A-3, A-4, A-6, D-1.
6. Publishing: B-1, B-2, B-3.
7. Reclaim and recovery: D-2, D-3, D-4, D-5, then weight: C-3.
8. Automate the regtest walk-through. This is the regtest gate.
9. Signet: E-4, E-5, E-3, then E-1 and E-2, then the board: C-5, C-6, B-4, and D-6.
10. Draft NIP and outreach to two projects that display offers. This is the signet gate, with the tester and soak targets.
11. Mainnet beta: F-3, F-4, F-5, then the rehearsal bond.

Steps 1 to 3 need no node and no relay, and they contain every line of code that can lose money. Spend the most time there.
