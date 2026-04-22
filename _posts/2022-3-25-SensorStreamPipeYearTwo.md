---
layout: post
title: "Sensor-Stream-Pipe, year two: OAK-D + RAAS integration"
---

Follow-on to the [2021 SSP post]({% post_url 2021-10-20-SensorStreamPipeWindowsPort %}). Feb–Mar 2022 was the OAK-D + RAAS integration phase on the `oakd-raas` branch of [moetsi/Sensor-Stream-Pipe](https://github.com/moetsi/Sensor-Stream-Pipe).

## What landed

- **On-device FP16 pose → C++ parser**. The OAK-D runs the pose model on its Myriad X VPU and streams FP16 output. That output had to be parsed on the host in C++ and fed into the rest of the SSP pipeline. "linking on device/FP16 output to C++ parse pose" is the commit title, and that one line covers a few days of alignment-vs-struct-layout-vs-endian debugging.
- **ZMQ DRAFTS APIs**. Added a draft-API-using client/server path for the multi-socket shapes that the stable ZMQ APIs couldn't express cleanly. Marked DRAFTS in the commit because they can change between libzmq releases — pinning is load-bearing.
- **Body reader**. A body-reader component that coped with partial frames (what the upstream was calling a "dummy body reader, with limitations"). Shipped with a known limitation rather than a fake completeness.
- **Heartbeat / throttle**. `hb` (heartbeat) commits, plus `less throttle`, `removing mandatory throw used for testing recovering capacity`. Tuning the flow-control side of the pipeline so a fast sensor didn't OOM the receiver.
- **Image aspect ratio**. "tentative fix for image 'aspect' ratio" — the pipeline had been silently stretching one sensor's output because the metadata frame described the wrong dimensions.
- **Diagnostic tools**. `diag tools` — small CLI helpers to inspect the stream at each stage, which is the boring work that makes everything else faster to debug.

## Integration-of-two-branches is its own category of work

The commit cadence through Feb–Mar 2022 is a long series of `fix / fix / fix / merge`. That's honest — when you're integrating two feature branches of someone else's project and the upstream is moving, you're not writing new code most days, you're keeping two moving trees coherent. It's the sort of work that doesn't demo well and is load-bearing anyway.

Contributions @ [moetsi/Sensor-Stream-Pipe](https://github.com/moetsi/Sensor-Stream-Pipe) (`oakd-raas` branch).
