+++
title = "MoldUDP64 TotalView-ITCH Replay Server"
date = "2025-07-22"
description = "Sample article showcasing basic Markdown syntax and formatting for HTML elements."
tags = [
    "C++",
]
+++


This post covers my implementation of a MoldUDP64 server that replays TotalView-ITCH market data. You can control replay speed and starting market phase while preserving the original message timing. 

Instructions for building and running as well as requirements are in the [README.md](https://github.com/jamisonrobey/itch_mold_replay/blob/main/README.md).

## Context

### Why / Use Cases

I wanted to write an [order book](https://en.wikipedia.org/wiki/Order_book) but needed realistic market data to test it properly. You can use this to test order book performance locally, with replay speed control for stress testing message ingress.

This also gave me experience with protocols used by exchanges.

### Protocols

- [MoldUDP64](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf):
    - UDP multicast protocol that sends sequenced packets with implicit message numbering  

    - Retransmission server for replaying missed packets by sequence number and count.
        -   Retransmission request includes sequence number of message and message count.

- [TotalView-ITCH](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf)
    - Binary market data format. For replay purposes, we only need each message's length and timestamp.


## Architecture

### Downstream

The [`Downstream_Server`](https://github.com/jamisonrobey/itch-mold-replay/blob/main/src/server/downstream_server.h) class handles the downstream feed by sequentially reading the file and packing ITCH messages into MoldUDP64 packets. It calculates when to send each packet by scaling the time elapsed since the first message:

```cpp
if (replay_ctx_.first_timestamp.count() == 0)
{
    replay_ctx_.first_timestamp = replay_ctx_.current_timestamp;
    return;
}
const std::chrono::nanoseconds elapsed{replay_ctx_.current_timestamp 
                                      - replay_ctx_.first_timestamp};
const auto delay{replay_ctx_.replay_start_time + (elapsed / replay_ctx_.speed)};
```
The delay calculation takes the elapsed time since the first message timestamp and divides by the replay speed to get the scaled delay. This is then added to the replay start time to get the absolute wall-clock time to transmit the packet. The timestamps are extracted as nanoseconds since midnight from each ITCH message (a feature in TotalView).

### Retransmission 

The [`Retransmission_Server`](https://github.com/jamisonrobey/itch-mold-replay/blob/main/src/server/retransmission_server.h) handles gap fill requests from clients who missed packets. It uses `SO_REUSEPORT` (great short overview available [here](https://lwn.net/Articles/542629/)) to distribute incoming UDP requests across multiple worker threads via [`Retransmission_Worker`](https://github.com/jamisonrobey/itch-mold-replay/blob/main/src/server/retransmission_worker.h) instances, each running their own epoll event loop. Workers monitor a shared `shutdown_fd` in their epoll set, which is written to by the retransmission server, workers detect the event and complete any in-progress transmissions before exiting.

### Synchronizing Downstream and Retransmission  

The downstream and retransmission threads need to be synchronized for two reasons. Firstly, retransmission workers must validate requested sequence numbers are valid (not in the future). Secondly, the downstream server only assigns sequence numbers as it processes messages. To handle arbitrary retransmission requests, we need a way to map sequence numbers back to their corresponding messages in the file.

#### Message Buffer

Since messages come from a read-only file and we need to map sequence numbers to file offsets, we use a ring buffer([`Message_Buffer`](https://github.com/jamisonrobey/itch-mold-replay/blob/main/src/server/message_buffer.h) class) to maintain a sliding window of recent messages with a lock-free write index. 
Retransmission threads can get the offset for any message given a sequence number efficiently because sequence numbers are always unique, making `sequence_num % buffer_size`
a perfect hash for indexing.


##### How to Choose Message Buffer Size

First, choose a power of 2 for fast modulo operations (bit masking instead of division).

The actual size depends on your desired retransmission window. Here are some statistics from testing with a NASDAQ ESM TotalView file to help decide:

The average message length was ~30 bytes. Including the 2-byte length prefix per message in MoldUDP64, each message consumes ~32 bytes on average. Each downstream packet has a maximum size of 1300 bytes, we use 1300 instead of the full 1500-byte MTU to account for IP/UDP headers and leaving overhead in case for VPNs etc that might add data to frames. With the 20-byte MoldUDP64 header, this leaves 1280 bytes for messages, giving us a rough estimate of (a feature in TotalView):
```1280 / 32 = 40 messages per packet```

At market open, the server sent ~20,000 packets/second, resulting in:

`40 × 20,000 packets/sec = 800,000 messages/sec`

The buffer size is set to `1 << 22` (`2^22` entries which is about 4.2 million). Each entry is 16 bytes, so this consumes ~67MB of memory. This provides a retransmission window of:

`4,200,000 entries / 800,000 messages/sec ≈ 5.25 seconds`

This window grows as market activity slows. This should be plenty of time for retransmission but you can adjust the size if desired by modifying the buffer size in [`config.h`](https://github.com/jamisonrobey/itch-mold-replay/blob/main/src/constants/config.h).