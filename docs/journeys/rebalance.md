# rebalance

grant a scoped agent permission to manage a uniswap LP position. the agent can rebalance when the position drifts out of range — but it cannot move tokens, transfer approvals, or do anything else. the scope is enforced on-chain by eip-8141 verify frames.

## summary

a user holds a concentrated LP position that needs active management. running their own bot requires technical skill and uptime. giving a third-party agent a private key means full access to everything in the wallet.

grantr solves this with a scoped session key. the agent operates autonomously within boundaries the user configures. the chain enforces the boundaries, not the agent.

**scenario:** user has an LP position in uniswap. authorizes an agent. grants a session key scoped to rebalance only — enforced on-chain via eip-8141 verify frames. LP drifts out of range. agent detects it, builds a frame tx, rebalances. agent tries to call `transfer` — rejected, selector not in policy. user revokes the session — agent's next tx fails.

one demo, four scenarios: the permissions layer eip-8141 is missing.

## user journey

| step | user | app |
|---|---|---|
| 1 | opens profile | sees balance, sparkline, tiles |
| 2 | taps + | action sheet opens |
| 3 | picks rebalance | positions screen loads |
| 4 | taps out-of-range position | authorize screen |
| 5 | configures policy | agent · spending limit · access window · keep-running toggle |
| 6 | taps authorize | tx preview shows frame 0 · frame 1 · nullifier · gas sponsored |
| 7 | reviews and signs | lifecycle: signing → submitted → confirmed |
| 8 | lands on dashboard | active session card, countdown, "monitoring your position" |
| 9 | watches agent work | feed narrates: price moved → out of range → rebalancing → rebalanced → earning |
| 10 | sees job done nudge | "agent finished · safe to remove" (if keep-running off) |
| 11 | taps remove agent | confirm dialog → tx preview (revokeSession) → signs → removed |

## how it works

```
user                       grantr                     chain
 │                           │                          │
 │─tap authorize──────────→ │                          │
 │                           │─grant session─────────→ │ frame 0 verify ✓
 │                           │                          │ frame 1 execute ✓
 │                           │                          │
 │                           │ ←agent rebalances───── │ frame 0 verify ✓
 │                           │                          │ frame 1 execute ✓
 │                           │                          │
 │                           │ ←agent transfer()───── │ frame 0 verify ✗ rejected
 │                           │                          │ (frame 1 never runs)
 │                           │                          │
 │─revoke session──────────→ │─session invalidated──→ │
```

every transaction the agent submits is a frame tx. frame 0 (verify) checks that the called function is in the user's allowlist and the session key is still valid. if yes, frame 1 (execute) runs the action. if no, the whole tx reverts on-chain — before anything happens.

## session policy

### user configures

| field | default | options | note |
|---|---|---|---|
| agent | grantr agent | grantr agent (recommended) · custom address | custom address shows validation warning if contract cannot be verified |
| spending limit | 2.0 ETH | any value > 0 | max any single action can move |
| access window | 1h | 1h · 6h · 24h · 7d | session expires automatically when window ends |
| keep running after rebalance | off | on · off | off = one rebalance then stops. on = loops until revoked or expired |

### fixed by grantr

| field | value | note |
|---|---|---|
| allowed contract | auto from selected position | the uniswap v4 position manager for the user's position |
| allowed actions | remove liquidity · claim fees · add liquidity | maps to `decreaseLiquidity` · `collect` · `increaseLiquidity` selectors |
| gas | sponsored by grantr | paid via eip-8141 paymaster · user pays 0 ETH |
| chain | etherex · devnet | fixed for v1 |

## agent visibility

the dashboard feed narrates the agent's activity in real time. each state renders as a feed entry with a status dot and optional frame chips.

| state | feed entry | text | frame chips |
|---|---|---|---|
| idle | monitoring dot (no entry) | monitoring your position | — |
| preparing | monitoring dot amber | agent preparing rebalance... | — |
| price moved | info (gray) | price moved · WBTC/ETH now at X · drifting toward edge of range | — |
| out of range | info (gray) | position out of range · not earning fees until rebalanced | — |
| rebalanced | success (green) | agent rebalanced · tx ... · new range: X – Y · gas: 0.002 ETH | frame 0 verify ✓ · frame 1 execute ✓ |
| earning | info (gray) | position earning · back in range · earning $X/hour | — |
| blocked | blocked (red) | your agent tried to [action] | frame 0 verify ✗ rejected · frame 1 skipped |
| job done (one-shot) | info (gray) | agent finished its one rebalance · safe to remove | — |
| session expiring (5 min) | card turns amber | — | — |
| session expiring (1 min) | card turns red | — | — |
| session expired | info (gray) | the time window you set ran out · authorize a new agent to continue | — |
| session revoked | blocked (red) | agent removed — session revoked | — |

tap any feed entry to expand a detail card showing: action · range (if rebalance) · gas used · block · tx hash. for blocked entries: attempted action · reason · allowed actions · enforced by · result.

## feedback

| moment | toast | color |
|---|---|---|
| agent authorized | agent authorized | green |
| signing rejected by user | signing rejected | red |
| authorization failed | authorization failed | red |
| agent rebalanced | rebalanced | green |
| agent blocked | agent blocked — unauthorized action | red |
| session expires in 5 minutes | session expires in 5 minutes | amber |
| session ending in under 1 minute | session ending soon | red |
| session expired | session expired | neutral |
| agent finished one-shot | agent finished · safe to remove | green |
| agent removed | agent removed | green |
| removal failed | removal failed | red |
| not enough ETH for gas | not enough ETH for gas | amber |

## constraints

| constraint | behavior |
|---|---|
| one session at a time | cannot authorize a new agent while one is active · must revoke first |
| no editing active sessions | to change policy, revoke and re-authorize |
| no ETH required for gas | gas sponsored by grantr via eip-8141 paymaster · user pays 0 ETH |
| insufficient ETH warning | if wallet has no ETH at all, amber warning appears on authorize screen · action is blocked |
| authorization tx failure | if tx reverts, red toast + failure modal with "try again" option |
| user can reject signing | on any tx preview, tapping reject cancels and returns to prior screen · no state changes |
| one-shot mode default | keep-running is off by default · agent stops after one successful rebalance |
| session auto-expires | when access window runs out, session transitions to removed state automatically · no user action needed |
| custom agent warning | pasting a custom agent address shows a validation warning if the contract cannot be verified as a valid agent |
| empty positions | if wallet has no uniswap v4 LP positions on etherex, positions screen shows empty state with link to uniswap |
| non-custodial | grantr never holds user funds · all actions go through the user's own wallet |
| etherex only | v1 supports etherex devnet only · no mainnet, no other chains |
| uniswap v4 only | v1 supports uniswap v4 positions only · no curve, aerodrome, or balancer |

## language

how spec concepts map to what the user sees.

| spec term | consumer term |
|---|---|
| scoped permissions | what the agent can do |
| session key | agent access |
| grant session | authorize agent |
| revoke session | remove agent |
| allowed selectors | allowed actions |
| session policy | agent rules |
| verify frame rejected | agent blocked |
| session expired | access expired |
| activity feed | agent history |
| max value per tx | spending limit |
| duration | access window |
| keep-running flag | keep running after rebalance |
| frame 0 (verify) | verify step |
| frame 1 (execute) | onchain action |
| paymaster sponsorship | gas paid by grantr |
| tx pending | submitted · waiting for block |
| tx confirmed | confirmed |
| user rejected signature | signing rejected |
| insufficient gas | not enough ETH |
| position drift | price moved · drifting toward edge |
| out of tick range | position out of range · not earning |
| in tick range | position earning |
| rebalance (decrease + collect + increase) | rebalanced |
| frame transaction | (not shown to user) |
| approve opcode | (not shown to user) |
| framedataload | (not shown to user) |
