[![Status: work in progress](https://img.shields.io/badge/status-work%20in%20progress-orange)]()

# grantr

## 1. summary

grantr is a frames-native account, with an app built to manage it. as a reference implementation for eip-8141, it covers account creation, recovery, delegation, and multi-device — not swaps, bridging, or the rest of a full financial product. no seed phrase, no lost accounts, no blanket approvals. your passkey signs, your address is permanent, and every permission you grant is enforced by the chain — not by an app asking nicely. open source, documented, and intended as a guide for developers and designers building on frames. try the prototype: https://alexanderchopan.github.io/grantr/prototype/grantr-prototype.html

## 2. references

- [eip8141.io](https://eip8141.io) is the protocol itself — opcodes, frame modes, and the rationale behind the design. [demo.eip-8141.ethrex.xyz](https://demo.eip-8141.ethrex.xyz) is the existing frame transaction demo, with generic examples like gasless approve+swap and LP rebalance with USDC gas payment. in both, the frames are the main character. grantr is a wallet built on frames where the user is the main character — onboarding, recovery, delegation, and multi-device policies, all surfaced through consumer ux.
- **what grantr exercises from the spec:** `VERIFY` frames with custom signature schemes (passkeys via P-256), `APPROVE` for execution and payment (sponsor pays user's gas), session keys as scoped `VERIFY` frames, key rotation via `DEFAULT` frame post-ops, and atomic batching for recovery and multi-device flows.
- **what grantr doesn't cover:** post-quantum signatures, the mempool model, cross-chain frames, non-wallet use cases.

## 3. the problem

**user:** anyone who holds crypto. not just power users.

most crypto accounts today are EOAs — your address IS your private key, and the seed phrase is its only backup. smart accounts soften this with recovery and passkey signing, but each wallet implements these features differently and they don't interoperate. and whatever wallet you use, authorizing an agent or a dapp gives it too much power — the wallet can only ask nicely, the chain doesn't enforce what you actually agreed to.

**five things grantr lets users do that today's wallets can't:**

| user wants | how it works today | what happens | problem |
|---|---|---|---|
| hold crypto without protecting a private key yourself | every wallet gives you a private key (12/24 words) that IS your account | you lose track of it or someone finds it and drains your account | the private key is both the credential and the only backup |
| get back in when you lose your phone | if you backed up the seed phrase, you regenerate the key. if not, gone. | phone breaks, no backup, account gone | recovery depends on the user having safely stored a backup |
| keep the same account after recovery | recovery gives you a new address. history, nfts, ens stay on the old one. | you recover but lose everything tied to the old address | one key, one address. rotating the key means abandoning everything |
| let an agent do a specific thing without full control | hand over your key or approve a token contract for unlimited amounts | agent gets hacked, everything approved can be moved | the wallet shows prompts but the chain doesn't enforce what you agreed to |
| require two devices for big transactions | set up a multisig in a separate app (safe) | skip it (one device = full risk) or run a second app | requiring two signers isn't a wallet feature — it's a workaround |

## 4. the solution

grantr addresses the five problems through frame policies on a single account. each policy is demonstrated in its own journey:

| journey | what it demonstrates | frame policy |
|---|---|---|
| **email recovery** (default) | creating an account with a passkey, recovering via verified email | passkey `VERIFY` + grantr attestation for rotation |
| **with guardians** | adding human guardians, recovering through them without email | passkey `VERIFY` + guardian-quorum policy for rotation |
| **with multi-device** | adding a second device, threshold for high-value txs | `VERIFY` with threshold over multiple passkeys, conditional on value/selector/destination |
| **with delegation** | granting an agent scoped permissions the chain enforces | session key as a scoped `VERIFY` frame |

a user can move between these — or combine them — without changing wallets. only the frame policy on their account changes.

**a note on custody and email recovery:** the default setup uses email to make recovery familiar. grantr runs a small service that verifies email control and issues a signed attestation for key rotation. signing, sending, and delegation are fully non-custodial. recovery depends on grantr. users who want full independence can add guardians, a second device, or both — email recovery can then be removed entirely.

**a note on smart accounts:** most of what grantr does is possible today in some smart wallet — passkey signing, social recovery, device thresholds, session keys — but each is a bespoke implementation. grantr makes them frame policies on a shared primitive. switching setups is a policy update, and any frames-compatible client can interact with the account the same way.

## 5. user journey — with email recovery (default)

one passkey signs; a verified email authorizes recovery if the device is lost.

| step | user | app |
|---|---|---|
| 1 | opens the app | welcome screen: "create account" / "recover account" |
| 2 | taps create account | prompts for email |
| 3 | enters email | sends verification link |
| 4 | back in app, prompted for passkey | "use your device to secure your account" |
| 5 | confirms with face id / touch id | device generates P-256 keypair in secure enclave |
| 6 | sees "creating your account..." | submits account-creation frame tx (pending → submitted → confirmed) |
| 7 | lands on home | address, zero balance, email recovery badge |
| 8 | loses phone, opens grantr on new device | taps "recover account" |
| 9 | prompted for email | grantr sends verification link |
| 10 | clicks link, back in app | creates new passkey on new device |
| 11 | grantr submits recovery tx | frame 0: verify email attestation · frame 1: swap in new passkey |
| 12 | lands on home | same address, same balance, same history — access restored |

setup fee covered by grantr. account upgradeable to guardians or multi-device at any time.

## 6. user journey — with guardians

alice adds guardians so recovery doesn't depend on grantr. guardians can only co-sign a key rotation — they can't sign transactions or see activity.

### adding guardians

| step | user | app |
|---|---|---|
| 1 | taps "add guardians" | explains what guardians do |
| 2 | picks 3 guardians, 2-of-3 quorum | — |
| 3 | invites each by ens (primary) or email (fallback) | generates invite with one-time code |
| 4 | each guardian accepts, creates passkey | sends public key to grantr |
| 5 | all confirmed | shows full list, prompts confirmation |
| 6 | alice confirms | tx preview: add guardian policy · verify (passkey) · execute (update policy) |
| 7 | signs with passkey | submits policy-update frame tx |
| 8 | lands on home | guardians badge active |

### recovering via guardians

| step | user | app |
|---|---|---|
| 1 | loses phone, opens grantr on new device | taps "recover account" |
| 2 | enters address or ens | grantr detects guardian policy |
| 3 | shown recovery options | "2-of-3 guardian quorum must approve" |
| 4 | creates new passkey | new P-256 keypair in secure enclave |
| 5 | sends recovery request to guardians | guardians notified via ens / email |
| 6 | two guardians approve | each signs with own passkey (VERIFY frames) |
| 7 | quorum met | key-rotation frame tx: verify quorum → execute swap |
| 8 | lands on home | same address — access restored |

## 7. user journey — with multi-device

alice adds her laptop as a second signer. the laptop creates a NEW passkey distinct from the synced copy.

### adding a second device

| step | user | app |
|---|---|---|
| 1 | taps "add device" on phone | prompts to open grantr on laptop |
| 2 | signs in on laptop with synced passkey | detected as new device: "register as second signer?" |
| 3 | creates new passkey on laptop | new P-256 keypair, distinct from phone's |
| 4 | laptop sends public key | phone prompted to confirm |
| 5 | confirms | — |
| 6 | sets threshold | "above $1,000: both devices required" |
| 7 | confirms | tx preview: add device + threshold policy |
| 8 | signs with phone passkey | submits policy-update frame tx |
| 9 | both devices show account | device indicator and threshold visible |

### signing a large transaction

| step | user | app |
|---|---|---|
| 1 | initiates tx above threshold on phone | tx preview: requires both passkeys |
| 2 | signs with phone | laptop receives co-sign notification |
| 3 | phone shows "waiting for laptop" | — |
| 4 | opens grantr on laptop | same tx preview: "you are the second signer" |
| 5 | signs with laptop | frame tx submitted with both signatures |
| 6 | confirmed on both devices | activity updates simultaneously |

## 8. user journey — with delegation

alice grants a scoped session key to an agent. the chain enforces the scope.

### granting a session

| step | user | app |
|---|---|---|
| 1 | accepts agent request (walletconnect-style) | shown: agent name, contracts, selectors, limits, duration |
| 2 | reviews and adjusts limits | can tighten caps, shorten duration, narrow selectors |
| 3 | taps authorize | tx preview: grant session key · verify (passkey) · execute (add session key) |
| 4 | signs with passkey | submits grant frame tx |
| 5 | lands on home | active session shown with scope and limits |

### the agent acts (and is bounded)

| step | actor | chain |
|---|---|---|
| 1 | agent submits allowed action | verify (session key) ✓ → execute ✓ |
| 2 | alice sees success in activity | — |
| 3 | agent submits action outside scope | verify (session key) ✗ — rejected |
| 4 | alice sees blocked entry in activity | shows what was tried and why it failed |

### revoking a session

| step | user | app |
|---|---|---|
| 1 | taps agent in sessions list | full session detail: scope, history, time remaining |
| 2 | taps revoke | tx preview: revoke session key |
| 3 | signs with passkey | submits revoke frame tx |
| 4 | session removed | agent's future frame txs fail at verify |

## 9. screens

five tabs with persistent balance bar. see prototype for complete screen set.

| tab | question | purpose |
|---|---|---|
| profile | who | sparkline activity chart + 2×2 tiles (security, signing, sessions, account). tile/list toggle. tap tile → bottom tray with detail. |
| sessions | what | frame receipt cards per session. blue dots for active, gray expired, red revoked. frame policy rows inline. divider between header and frames. |
| ⬩ (action) | — | send, receive, scan |
| activity | when | card per entry with stacked frame rows. dots (6px): green pass, red fail. filter pills: own · guardians · agents · sponsors · contracts · sent. white active pill. |
| addresses | who/where else | full directory. 2×3 stacked filter pills. contextual role pills (guardian shows quorum position, agent shows session status). search bar. |

## 10. feedback

~30 toasts organized by journey. green = success, red = failure/blocked, amber = warning, neutral = informational. 


## 11. user flow diagrams

superseded by prototype. 

## 12. constraints

**in:** four journeys, passkey signing (P-256), key rotation, session keys, multi-device thresholds, sponsored gas, frame-visible tx preview, activity feed with frame detail.

**out:** multi-chain, swaps/bridging, portfolio management, push notifications, fiat ramp, post-quantum, mempool handling, cross-chain.

**limited:** one account per device, ens + email for guardians, single example agent, no social features.

**not in scope:** contract audit, regulatory.

---

## design language

**palette:** #000 bg, #0a0a0a cards, #22c55e green (brand/pass/balance), #ef4444 red (fail/blocked), #f59e0b amber (guardians/warnings), #3b82f6 blue (sessions/agents), #fff primary text.

**typography:** DM Sans for UI, DM Mono for addresses and frame labels. all lowercase.

**dots:** 6px green = pass, red = fail. 8px blue = active session, gray = expired, red = revoked.

**components:** white active pills (not green). cards with 14px radius. bottom trays (cash app pattern). filled tab icons when active, outlined when inactive.

**references:** cash app (tab bar, balance, trays), apple mail (dot status indicators), linear (feed patterns, dividers, dark mode).

---

## prototype

### demo journeys

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
