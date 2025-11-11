+++
title = "Nasdaq MoldUDP64 TotalView-ITCH Feed Simulator" 
date = "2025-07-22" 
lastmod = "2025-11-11"
description = "Post on architecture and design decisions for Nasdaq MoldUDP64 TotalView-ITCH feed simulator in C++" 
tags = [
    "C++",
]
+++ 

This post covers my implementation of a MoldUDP64 server that replays TotalView-ITCH market data. You can control replay speed and the starting market phase, while preserving the original message timing.

Instructions for building and running as well as requirements are in the [README.md](https://github.com/jamisonrobey/nasdaq-moldudp64-feed-sim/blob/main/README.md).

## Context

### Why / Use Cases

I wanted to write an order book but needed realistic market data to test it properly. You can use this to test order book performance locally, with replay speed control for stress testing message ingress.

This also gave me experience with protocols used by exchanges.

### Protocols

- [MoldUDP64](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf):
    - UDP multicast protocol that sends sequenced packets with implicit message numbering (downstream)
    - Retransmission server replays missed packets by sequence number and count. A retransmission request includes the starting sequence number and the number of messages to resend.

- [TotalView-ITCH](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf)
    - Binary market data format. For replay purposes, we only need each message's length and timestamp.

## Architecture

### Downstream

[`DownstreamServer`](https://github.com/jamisonrobey/nasdaq-moldudp64-feed-sim/blob/main/src/downstream_server.h) handles the downstream feed by sequentially reading the file and packing ITCH messages into MoldUDP64 packets. It calculates when to send each packet by taking the elapsed time since the first message timestamp, dividing by the replay speed to get the scaled delay. This is then added to the replay start time to get the absolute wall-clock time to transmit the packet. 

### Retransmission 

[`RetransmissionServer`](https://github.com/jamisonrobey/nasdaq-moldudp64-feed-sim/blob/main/src/retransmission_server.h) handles gap fill requests from clients who missed packets. It uses `SO_REUSEPORT` (great short overview available [here](https://lwn.net/Articles/542629/)) to distribute incoming UDP requests across multiple worker threads via [`RetransmissionWorker`](https://github.com/jamisonrobey/nasdaq-moldudp64-feed-sim/blob/main/src/retransmission_worker.h) instances, each running their own epoll event loop. Workers monitor a shared `shutdown_fd` in their epoll set. When the retransmission server writes to it, workers detect the event and complete any in-progress transmissions before exiting.

### Synchronizing Downstream and Retransmission  

The downstream and retransmission threads need to be synchronised for two reasons. Firstly, retransmission workers must validate requested sequence numbers are valid (not in the future). Secondly, the downstream server only assigns sequence numbers as it processes messages. To handle arbitrary retransmission requests starting from any sequence, we need a way to map sequence numbers back to their corresponding messages in the file.

#### Message Buffer

Since messages come from a read-only file and we need to map sequence numbers to file offsets, we use [`MessageBuffer`](https://github.com/jamisonrobey/nasdaq-moldudp64-feed-sim/blob/main/src/message_buffer.h) to maintain a sliding window of recent messages with a lock-free write index. Retransmission threads can get the offset for any message given a sequence number efficiently because sequence numbers are always unique, making `sequence_num % buffer_size` a perfect hash for indexing.

##### How to Choose Message Buffer Size

One of the optional parameters is the retransmission buffer size so this section is to give a rough idea on what size you might want. 

- First, choose a power of 2 for fast modulo operations (bit masking instead of division), the constructor will throw if you don't anyway. The size then depends on your desired retransmission window. 
- Here are some statistics from testing with a NASDAQ ESM TotalView file to help decide:

    - The average message length was ~30 bytes. Including the 2-byte length prefix per message in MoldUDP64, each message consumes ~32 bytes on average. Each downstream packet has a maximum size of 1300 bytes, we use slightly less than the MTU here because we want to leave room for any extra info on the packets (VPNs etc). With the 20-byte MoldUDP64 header, this leaves 1280 bytes for messages, giving us a rough estimate of:

        - `1280 / 32 = 40 messages per packet`
        - At market open, the server sent ~20,000 packets/second, resulting in:
        - `40 × 20,000 packets/sec = 800,000 messages/sec`
        - The default buffer size is `1 << 22` (`2^22` or ~= 4.2 million). 
            - Each entry is 16 bytes, so this would use ~67MB of memory and provides a retransmission window of `4,200,000 entries / 800,000 messages/sec ≈ 5.25 seconds`
