grantr | frames-native smart account · eip-8141 reference implementation

# [grantr](https://alexanderchopan.github.io/grantr/)

![Status: work in progress](https://img.shields.io/badge/status-work%20in%20progress-orange)

# grantr

## 1. summary

grantr is a frames-native account, with an app built to manage it. as a reference implementation for eip-8141, it covers account creation, recovery, delegation, and multi-device — not swaps, bridging, or the rest of a full financial product. no seed phrase, no lost accounts, no blanket approvals. your passkey signs, your address is permanent, and every permission you grant is enforced by the chain — not by an app asking nicely. open source, documented, and intended as a guide for developers and designers building on frames.

**try the prototype:** [alexanderchopan.github.io/grantr](https://alexanderchopan.github.io/grantr/)

## 2. references

* [eip8141.io](https://eip8141.io) is the protocol itself — opcodes, frame modes, and the rationale behind the design. [demo.eip-8141.ethrex.xyz](https://demo.eip-8141.ethrex.xyz) is the existing frame transaction demo, with generic examples like gasless approve+swap and LP rebalance with USDC gas payment. in both, the frames are the main character. grantr is a wallet built on frames where the user is the main character — onboarding, recovery, delegation, and multi-device policies, all surfaced through consumer ux.
* **what grantr exercises from the spec:** `VERIFY` frames with custom signature schemes (passkeys via P-256), `APPROVE` for execution and payment (sponsor pays user's gas), session keys as scoped `VERIFY` frames, key rotation via `DEFAULT` frame post-ops, and atomic batching for recovery and multi-device flows.
* **what grantr doesn't cover:** post-quantum signatures, the mempool model, cross-chain frames, non-wallet use cases.

## 3. the problem

**user:** anyone who holds crypto. not just power users.

most crypto accounts today are EOAs — your address IS your private key, and the seed phrase is its only backup. smart accounts soften this with recovery and passkey signing, but each wallet implements these features differently and they don't interoperate. and whatever wallet you use, authorizing an agent or a dapp gives it too much power — the wallet can only ask nicely, the chain doesn't enforce what you actually agreed to.

**five things grantr lets users do that today's wallets can't:**

| user wants | how it works today | what happens | problem |
| --- | --- | --- | --- |
| hold crypto without protecting a private key yourself | every wallet gives you a private key (12/24 words) that IS your account | you lose track of it or someone finds it and drains your account | the private key is both the credential and the only backup |
| get back in when you lose your phone | if you backed up the seed phrase, you regenerate the key. if not, gone. | phone breaks, no backup, account gone | recovery depends on the user having safely stored a backup |
| keep the same account after recovery | recovery gives you a new address. history, nfts, ens stay on the old one. | you recover but lose everything tied to the old address | one key, one address. rotating the key means abandoning everything |
| let an agent do a specific thing without full control | hand over your key or approve a token contract for unlimited amounts | agent gets hacked, everything approved can be moved | the wallet shows prompts but the chain doesn't enforce what you agreed to |
| require two devices for big transactions | set up a multisig in a separate app (safe) | skip it (one device = full risk) or run a second app | requiring two signers isn't a wallet feature — it's a workaround |

## 4. the solution

grantr addresses the five problems through frame policies on a single account. each policy is demonstrated in its own journey:

| journey | what it demonstrates | frame policy |
| --- | --- | --- |
| **email recovery** (default) | creating an account with a passkey, recovering via verified email | passkey `VERIFY` + grantr attestation for rotation |
| **with guardians** | adding human guardians, recovering through them without email | passkey `VERIFY` + guardian-quorum policy for rotation |
| **with multi-device** | adding a second device, threshold for high-value txs | `VERIFY` with threshold over multiple passkeys, conditional on value/selector/destination |
| **with delegation** | granting an agent scoped permissions the chain enforces | session key as a scoped `VERIFY` frame |

a user can move between these — or combine them — without changing wallets. only the frame policy on their account changes.

**a note on custody and email recovery:** the default setup uses email to make recovery familiar. grantr runs a small service that verifies email control and issues a signed attestation for key rotation. signing, sending, and delegation are fully non-custodial. recovery depends on grantr. users who want full independence can add guardians, a second device, or both — email recovery can then be removed entirely.

**a note on smart accounts:** most of what grantr does is possible today in some smart wallet — passkey signing, social recovery, device thresholds, session keys — but each is a bespoke implementation. grantr makes them frame policies on a shared primitive. switching setups is a policy update, and any frames-compatible client can interact with the account the same way.

## 5. prototype

the prototype is a single html file with a dev tools panel on the left and a mobile app on the right. nothing to install — open it in a browser.

**11 demo journeys, all wired:**

1. account creation
2. email recovery
3. add guardians
4. recover via guardians
5. add second device
6. co-sign large tx
7. grant delegation
8. agent acts / blocked
9. revoke delegation
10. rebalance portfolio
11. build transaction

**dev tools panel:**

- **tour** — a 6-step narrated walkthrough that threads through the whole product. start here if you've never seen grantr before.
- **journeys** — every one of the 11 journeys above wired as a single click.
- **actions** — deep-links to send / rebalance / build / sponsor / verify screens.
- **tabs** — profile / sessions / activity / addresses.

**on the phone:**

- tap ⊕ in the tab bar for the action sheet
- tap any active session card's **see it work** to run a live simulation — the agent attempts one allowed action (passes) and one out-of-scope action (blocked at the verify frame)
- tap any activity entry to expand its frame chain
- every tile, row, and pill is wired — nothing is decorative

**frames are the thesis. the prototype makes them visible wherever a transaction touches the ui:**

- **session cards** list their frame policy inline
- **activity entries** expand to show the frame chain of every tx — including blocked ones, with the failing frame marked
- **tx previews** show the full frame sequence before signing, not a single "approve" button
- **agent simulations** animate frames turning green (pass) or red (blocked) so you can watch the chain enforce the scope in real time

## 6. constraints

**in:** four journeys, passkey signing (P-256), key rotation, session keys, multi-device thresholds, sponsored gas, frame-visible tx preview, activity feed with frame detail.

**out:** multi-chain, swaps/bridging, portfolio management, push notifications, fiat ramp, post-quantum, mempool handling, cross-chain.

**limited:** one account per device, ens + email for guardians, single example agent, no social features.

**not in scope:** contract audit, regulatory.

## 7. status

- **feature-complete prototype** covering all four spec journeys plus the 11 demo journeys
- **not production software** — no real backend, no real etherex deployment, passkeys are simulated, accounts are mock state
- **future work**: connect to a real etherex devnet, backend for passkey registration + guardian invites, agent marketplace with real scoped agents, browser extension version

## 8. contributing

if you're building another eip-8141 reference implementation, an sdk, or a design system that borrows from this, take what's useful. prs and issues welcome.
