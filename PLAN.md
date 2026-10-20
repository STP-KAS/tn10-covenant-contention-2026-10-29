# Plan

Several senders spend one covenant state output at the same time. Only one spend can win each hop. The next hop spends the output that winner created. The phase measures four things, per minute:

- winners included on the virtual chain
- conflict rejects
- time from submit to inclusion, for winners only
- state hops

Attempt rate and hop rate are different. Extra attempts lose. They do not create extra hops.

## Nodes

The desk already runs one rusty-kaspa 2.1.0 node, Testnet-10, UTXO index on, default ram scale (no ram-scale flag). Leave that process up. Do not start a second kaspad on the desk. Do not wipe its app directory.

Its gRPC socket is reachable beyond the machine. A check while this plan was written found no established client on that socket. A local proxy holds the loopback gRPC port and forwards to the public bind. This phase does not need the public bind. Do not restart the node now. At the next restart, after the mini PC has taken over as the main node, stop that proxy first and bind gRPC to loopback only. Peer listen stays on loopback. Borsh stays on loopback. UPnP stays off. Default ram scale stays.

A mini PC node is expected around 22 Oct 2026. It becomes the main node when all of these are true: rusty-kaspa 2.1.0, Testnet-10, UTXO index on, default ram scale, synced, and the coordinator can read its virtual chain. Until then the desk node is the chain reader. After the handover the desk node stays up as one sender target.

Public submits use wRPC Borsh on Testnet-10 hostnames. The candidate set is muon-10.kaspa.blue, boson-10.kaspa.red, electron-10.kaspa.blue, neutrino-10.kaspa.stream, and vector-10.kaspa.green. Some of those names have returned the same figures and may be one backend. Before the phase, keep a public name only when its version is 2.1.0, its network id is testnet-10, and its mempool size is under 90,000. If two names move together, use one of them. A public node at or above 90,000 mempool entries is skipped for the rest of the phase. That guard is there because a ram-scale of 0.1 caps a pool at 100,000 and the next insert can panic the process.

Every connection aborts unless the network id is testnet-10.

## What is built

The contract is `contracts/public_mint.ag` in argent-lang/kcc20-reference at `c8a08711`. Kaspa wasm 2.1.0 can attach a covenant id to an output (`CovenantBinding`, `GenesisCovenantGroup`, `populateGenesisCovenants`). It does not compile this contract or build the redeem script. The reference build produces the genesis group, the redeem, and the next state script. Wasm signs the funding input and submits the finished transaction on Borsh. rusty-kaspa 2.1.0 has no `GetUtxosByCovenantIds`. That call is pull 1152, still open, and it needs a covenant index this release does not have. Do not rebuild kaspad to get it. Do not comment on either upstream repo.

The 7 Oct key is discarded. This phase uses a new key and a new launch path, so the covenant id is new. The builder refuses `launch::stp-trump2028` and `launch::stp-trump2028-last-call`.

Fixed parameters:

- version 1 transactions
- launch path `launch::contention-2026-10-29`
- extension is unkeyed BLAKE3 of the UTF-8 name `contention-2026-10-29`, which is `343d2caf126be774cb523fff4c60545879b8a407aeb1283f97ba8efb6e55af23`
- mint amount 1, and the per-mint limit is 1
- starting remaining 100,000
- owner scheme `0x00`, borrow scheme `0x00`, 32 zero bytes of borrow guard
- one fee rate for every sender, chosen at the start from the main node's normal bucket and floored at 100 sompi per gram

The relay fee is that rate times `max(compute mass, ceil(transient mass / 2))`. Storage mass is committed and is not the fee base. The 7 Oct mint was accepted with a redeem of about 6,000 bytes and a compute mass of about 8,700. Expect this hop in that range. If compute mass is above 100,000, do not submit.

Genesis spends one fresh funding coin. Output 0 is the minter and carries the covenant. Output 1 is plain change. The minter value is a few KAS and is copied unchanged onto every later minter output.

A hop is this shape, and every sender uses it:

- input 0 spends the current minter outpoint
- input 1 spends that sender's own funding coin
- output 0 is the next minter, same value, remaining reduced by 1
- output 1 is the token output, at the smallest value the node will relay
- output 2 is change back to that sender

The reference sets the compute budgets. The 7 Oct node accepted a covenant-input budget of 5 and a funding-input budget of 10. Keep whatever the reference emits. Sign and hash after mass and budgets are set.

Each sender holds its own funding coins, several thousand tKAS. A rejected attempt does not spend them. Only the winning hop spends a carrier and a fee. The fresh keys live outside this repo.

## How a hop advances

The coordinator watches the main node's virtual chain. The winner of the current outpoint is the transaction on that chain which spends it. The next state output is that transaction's output 0. Advance only after checking that output 0 has a covenant binding and the same KAS value as the old minter. A shape mismatch stops the phase.

The coordinator writes the current outpoint into a local file outside this repo. Senders read that file. They do not decide the next outpoint from their own submit acknowledgement. They do not ask public REST for coins. Both the minter and the token output can carry the same covenant id, so an id lookup would not by itself select the minter. The script-hash address also changes when `remaining` changes, which is why the missing covenant-id call does not block this plan.

Until a new outpoint is published, every sender keeps spending the published one. The first acceptance into a node's mempool wins that mempool. Later submissions of the same outpoint are the conflict rejects. Because the fee rate is the same on every sender, the race is arrival at each node, then which of those mempool copies a miner includes.

Do not submit the child before the parent is on the virtual chain. A chain of unconfirmed hops dies in the orphan pool. If a mempool acceptance has no virtual-chain inclusion within 30 seconds, write `STALL` in the phase log and stop.

## Senders

One process per node. Tags:

- `desk`, the desk Borsh socket
- `mini`, the mini PC, added only after it is the main node
- one tag per public name that survived the mirror check, using the first label (`muon`, `boson`, and so on)

The Thursday target is desk, mini, and up to three public names. The phase still runs if mini is late, as long as the drop gate has passed and at least two nodes are connected. The phase log records the tag set and the rate in the start line.

Each sender is a token bucket at the step's rate, aligned to the UTC second. One attempt is one fully built transaction handed to that sender's node.

## Rate steps

END is 2026-10-29T19:00:00.000Z. No new attempt is built at or after END.

| Step | UTC window | Attempts per second per sender |
| --- | --- | --- |
| 1 | 18:00:00 through 18:14:59.999 | 1 |
| 2 | 18:15:00 through 18:29:59.999 | 2 |
| 3 | 18:30:00 through 18:44:59.999 | 5 |
| 4 | 18:45:00 through 18:59:59.999 | 10 |

Genesis is included before 17:50 UTC. If it is not included by 17:55 UTC, the phase does not start.

At the top step, five senders produce 50 attempts per second. Earlier desk inclusion sat around a second, and one quiet mint landed inside the same second, so a hop rate of a few per second is the range to expect. The minute sheet is what decides it. Winners per minute should match hops per minute. Sent and ok will be larger.

## Logging

The run directory is outside this repo. Three files:

`logs/submit.jsonl` has one line per attempt, including losers. Fields, in this order:

- `ts`, UTC time to the millisecond, such as `2026-10-29T18:00:01.234Z`
- `sender`, the tag
- `node`, the same tag (the process and the node are one)
- `txid`
- `result`, `ok` or `rejected`
- `reason`, empty on `ok`, otherwise one of `already_spent`, `orphan`, `fee`, `mass`, `mempool_full`, `route`, `rpc`, `other`

Flush every line. The raw node sentence often contains transaction ids. Store the token above, not that sentence.

`logs/minute.csv` is counts only. One row per UTC minute. Header:

```text
minute_utc,step,sent,ok,rej_already_spent,rej_orphan,rej_fee,rej_mass,rej_mempool_full,rej_route,rej_rpc,rej_other,winners,hops,include_p50_ms,include_p95_ms
```

`sent` counts attempts. `ok` counts mempool acceptances. The `rej_*` columns split the rejects. `winners` counts our hops whose accepting-block header falls in that minute. `hops` counts state transitions observed from those accepting blocks. `include_p50_ms` and `include_p95_ms` are `accepting header time minus ts`, for winners whose `ts` falls in that minute. A minute with no winner leaves those two cells empty.

`logs/phase.log` is one short line per event, with the actual UTC second the event happened:

```text
2026-10-29T17:49:12Z GENESIS included
2026-10-29T18:00:01Z START senders=desk,mini,muon,boson step=1 rate=1
2026-10-29T18:15:00Z STEP step=2 rate=2
2026-10-29T19:00:00Z STOP reason=END
```

START and STOP use the second the coordinator actually began and ended attempts, which can differ from 18:00:00 and from END. STEP lines use the second the bucket changed. A stall or a shape mismatch is a STOP line with its own reason, at the second it was noticed.

Kaspa Pulse's count is transactions he sees in blocks. Compare his count with `winners`. `sent` and `ok` are local submit results. This plan does not message him.

A later commit on this repo may add `minute.csv` and `phase.log` only after a scan finds no key, seed, address, IP, or transaction id. `submit.jsonl` stays on disk.

## Stop

Two switches, both checked before every attempt.

The stop switch is a file `control/STOP` in the run directory. Creating it stops every sender. The phase log STOP line uses `reason=STOP` and the actual second.

The hard stop is END, 2026-10-29T19:00:00.000Z, even if `control/STOP` was never created. The STOP line then uses `reason=END`.

Whichever fires first is the one recorded. No sender restarts itself. An attempt already on the wire is logged, and nothing new is built.

## Risks

- A builder pointed at an old launch path would move a covenant this test does not own. The path allow-list is one path, and the key is new.
- Counting a mempool ack as a hop repeats an old mix-up with Pulse's block count. Hops move on the virtual chain only.
- Submitting the next hop before the parent is included produces orphan rejects and can stall the chain of state. The 30 second stall stop is the bound.
- Public nodes may reject the large redeem on fee or mass, or may be an older server. The desk test comes first. A node that rejects the shape is dropped from the tag set. If no public node accepts it, Thursday is desk and mini only, and the phase log says so.
- One copied transaction can sit in each mempool until a block. That is one entry per node per hop, not one entry per attempt. The 90,000 pause is still on.
- Relay throttling and "route is full" are their own reason codes. At these rates they should be rare. If `route` becomes the main reject, the step is still telling the truth and the phase keeps going until STOP or END.
- Remaining cannot run out inside the hour at mint amount 1 and a start of 100,000. A hop rate of 10 per second for the whole hour would use 36,000.
- Output order other than "output 0 is the next minter" makes the follower pick the token output. That is a shape mismatch and a stop, not a guess.
- The desk gRPC bind stays as it is until the planned restart. Nothing in this phase opens it further.
- A second kaspad on the desk collides with the running node. The mini PC is the other machine.

## By Tuesday 27 Oct 2026, 18:00 UTC

The drop rule is absolute. "Tested" means a real Testnet-10 submit, read back from the desk node's virtual chain: one fresh genesis, one included mint of amount 1, and one conflicting second spend rejected as already spent. A VM run does not count. If that trio is missing at the gate, add a one-line `DROPPED.md` on this repo and do not arm Thursday.

What I expect:

- The desk trio is realistic before the gate, and it is the right first test on Wednesday 21 Oct. The same node already accepted this contract shape on 7 Oct, wasm's missing compiler is already bypassed by the reference build, and the missing covenant-id call is bypassed by following output 0. Wednesday is desk only: three transactions, three submit lines, then stop. No rate loop and no public node.
- A second sender process on the desk, plus one public node accepting the same redeem, fits in the days after that if Wednesday passes. I want both before the gate. The drop rule does not wait for them.
- The mini PC is not part of the gate. If it is synced by Thursday it is the main node and a sender. If it is not, Thursday uses the desk as the chain reader.
- The four rate steps, the full tag set, and a comparison with Pulse are the Thursday phase. I do not expect them to be measured before the gate.

The thing most likely to slip is the first real redeem on this new path: a mass, fee, or script mismatch. A week is enough to correct that against the reference and retry on the desk. It is not enough to change contracts. If Wednesday's trio cannot be included, Thursday is dropped rather than debugged live.

## Out of scope

No mainnet. No spend of an existing covenant. No kaspad upgrade. No tag, release, or upstream comment. No code in this commit.
