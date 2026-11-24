---
sidebar_position: 2
title: First sync
description: "What happens during an orchestrator's first sync of an EVM network, how long it takes, how rate limits and retries behave, and the log lines to expect."
---

# First sync

On first start, and after a [backfill](backfill.md), the orchestrator scans every EVM network from the bridge contract's deployment block to the head, using inclusive ranges of up to `FilterQuerySize` blocks. On Ethereum mainnet that is a few thousand `eth_getLogs` queries, which is where free-tier providers push back.

## Before you start

1. Configure two or more independent, full-history endpoints per network. See [EVM networks](../networks.md).
2. Set `RpcRequestsPerSecond` and `RpcBurst` under the provider's degraded limit. See the [free-tier configuration example](../networks.md#low-rate-configuration) for fractional request rates and per-query block limits.
3. Make sure the Zenon node is synced. The orchestrator waits for it before doing anything else.

## What you will see

```
Sync progress for chainId 1: block 18004000 of 21003400 (24.9%), 500 ranges in 3m20s
```

A progress line appears every 25 ranges, followed by a completion line. Each successful range uses a head read and an `eth_getLogs` query.

At a fixed effective width `W`, the first range covers up to `W` blocks. Later multi-block ranges revisit the preceding range’s last block and cover up to `W - 1` new blocks. A one-block range advances by one block.

For a scan covering `N` blocks with both endpoints included, use this lower bound.

```text
ranges  = N                                   when W = 1
ranges  = 1 + ceil(max(0, N - W) / (W - 1))    when W > 1
minutes = ranges × 2 / RpcRequestsPerSecond / 60
```

Scanning from block 18,000,000 through 26,000,000 at 2,000 blocks per query takes 4,003 ranges. At 3 requests per second, that is about 45 minutes before retries or historical-event validation. At 8 blocks per range, the same scan takes about 1.14 million ranges. Automatic halving stops at 8 blocks. Explicit smaller widths from the operator or the provider are honored. See [EVM networks](../networks.md#rate-limits).

A transient provider error or timeout triggers a retry.

```
Sync for chainId 1: eth_getLogs for blocks [18004000, 18005999] failed at block 18004000 (attempt 3/12), retrying in 8s: 429 Too Many Requests
```

The same range is retried with backoff from 2 seconds to 2 minutes. After 12 consecutive failures the pass stops. What happens next depends on when it failed:

- **During the initial sync at startup**, the orchestrator exits. The supervisor restarts it (the systemd unit in [Install](../install.md) waits 10 seconds) and the sync resumes from the persisted cursor.
- **On a running node**, the next 3-minute refresh resumes from the same cursor.

A reply containing an out-of-range log retries the same range before applying any of its logs. The cursor **never** advances past a range that did not complete.

## Historical events are validated

For each mature unwrap, the signer checks the canonical block hash and confirmation depth across the configured endpoints. Each endpoint must also return the same bridge event from its receipt or a single-height log query. The signer skips an agreed orphan block. Missing or conflicting evidence causes the range to retry. A disagreement can appear in the log as follows.

```
cannot validate historical unwrap 0x…/3 against the canonical chain: canonical block 18004123 inconclusive, 1 of 2 endpoints did not answer
```

## After the scan

The orchestrator subscribes to new logs and refreshes the subscription every 3 minutes with a short catch-up sync. `latestUpdateHeight` in [`getStatus`](../health-api.md) should then follow the head.

Live unwraps wait in `queues/` until the same confirmation and transfer checks pass. The signer discards a queued event only after all endpoints report the same replacement block across three checks 30 seconds apart. Before a signing ceremony, the signer rechecks the selected unsigned records against the source event, including the recipient, token, amount and any affiliate split. This covers records created by an earlier binary or restored from a checkpoint. A batch with unresolved source evidence waits for a later ceremony.
