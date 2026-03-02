# feat: expose per-packet TWCC timing as Event::TwccFeedback

## Problem

str0m's built-in BWE (GCC/TWCC) works well for general use, but there is no way
for callers to plug in an **external bandwidth estimator**. The raw per-packet
timing data from TWCC feedback is processed internally and discarded — the caller
only ever sees the final aggregate `EgressBitrateEstimate`.

Use cases that need this:
- ML-based BWE models (e.g. Google's `rl4bwe`, Microsoft's R3Net) that require
  per-packet send/receive timestamps, OWD deltas, and loss patterns as input features
- Research and A/B testing where you want to compare your own estimator against
  the built-in one
- Applications that need raw TWCC data for their own telemetry/logging

## Solution

Add a new `Event::TwccFeedback(Vec<TwccPacketReport>)` event that fires **once per
TWCC RTCP report**, delivering the per-packet timing records to the caller before
the aggregate `EgressBitrateEstimate`.

`TwccPacketReport` carries:

| Field              | Type              | Description                                        |
|--------------------|-------------------|----------------------------------------------------|
| `seq`              | `u64`             | TWCC sequence number                               |
| `send_time`        | `Instant`         | When the sender dispatched the packet              |
| `local_recv_time`  | `Option<Instant>` | When the sender got the ACK (`None` = lost)        |
| `remote_recv_time` | `Option<Instant>` | Receiver-side arrival time from TWCC delta         |
| `size_bytes`       | `usize`           | RTP payload size in bytes                          |
| `is_probe`         | `bool`            | Whether it was a bandwidth-probe cluster packet    |
| `rtt`              | `Option<Duration>`| `local_recv_time - send_time` (`None` if lost)     |

## Design decisions

### 1. Zero impact on existing BWE

The implementation uses `.inspect()` — a lazy iterator adaptor — to collect records
as a side-effect while the iterator is passed unchanged to the existing `bwe.update()`.
The built-in BWE sees exactly the same data as before. No behavioural change for
existing users.

### 2. Works without BWE configured

If no BWE is attached, the iterator is consumed via `for_each(|_| {})` so
`TwccFeedback` still fires. This lets callers run their own estimator standalone
without enabling the built-in one.

### 3. Ordering guarantee

`TwccFeedback` is emitted from `poll_event()` *before* `EgressBitrateEstimate`.
This ensures external estimators can act on the raw data before the built-in
estimate is delivered — useful if the caller wants to override or blend estimates.

### 4. Additive, non-breaking

`Event` is `#[non_exhaustive]` so adding a new variant is not a semver break.
`TwccPacketReport` is a new public type — no existing API is changed.

## Files changed

| File | Change |
|------|--------|
| `src/bwe/api.rs` | New `TwccPacketReport` struct |
| `src/lib.rs` | New `Event::TwccFeedback` variant + trace arm in `poll_output()` |
| `src/session.rs` | `.inspect()` collection in `handle_feedback()`, drain in `poll_event()` |

## Example usage

```rust
while let Ok(Output::Event(event)) = rtc.poll_output() {
    match event {
        Event::TwccFeedback(reports) => {
            for pkt in &reports {
                my_bwe_model.on_packet(
                    pkt.seq,
                    pkt.send_time,
                    pkt.remote_recv_time,
                    pkt.size_bytes,
                );
            }
            let estimate_bps = my_bwe_model.estimate();
        }
        Event::EgressBitrateEstimate(bwe) => {
            // built-in BWE still works unchanged
        }
        _ => {}
    }
}
```
