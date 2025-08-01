---
title: Flow Control Across Network Stacks - TCP, HTTP/2, and Reactive Streams
summary: Flow Control Across Network Stacks
date: 2020-06-07
authors:
  - admin
tags:
  - network
---

## Introduction

Let’s suppose we are building a network application using a reactive stream implementation. 

Reactive streams provide flow control through a mechanism known as *back-pressure*. A subscriber in a reactive stream can signal availability to the connected publisher, thereby controlling the size of streamed data. However, there are additional flow control mechanisms behind at both the application layer (HTTP/2) and the transport layer (TCP).

That means, when streaming data to an end system, these layers of flow control are at play. In this post, we’ll explore each of these layers in order—starting from the lowest level.

## TCP Layer

It’s well-known that TCP provides flow control. On the client side, it offers a way to control how many TCP segments can be received at once. The client checks the available *window size* in the TCP header for the connected end system.

![tcp\_congestion\_control](https://github.com/taehyeok-jang/taehyeok-jang.github.io_legacy/assets/31732943/a92b360a-2a28-4636-9213-c84b077e37bf)

During the initial TCP 3-way handshake, both ends initialize their *receive window size (rwnd)* based on system defaults. If one side becomes overwhelmed, it can advertise a smaller window size. A window size of zero means that no further data can be sent until the receiver has drained its application buffer. This process continues throughout the life of the TCP connection. Each ACK includes the latest rwnd value, enabling both sender and receiver to dynamically adjust the data flow based on their capacity and processing speed.

The sender transmits data only if:

```
(LastByteSent - LastByteAcked) < rwnd
```

The receiver continuously advertises its available buffer size through the TCP header's window field:

```
rwnd = RcvBuffer - (LastByteRcvd - LastByteRead)
```

## HTTP/2

![fig7\_1\_alt](https://github.com/taehyeok-jang/taehyeok-jang.github.io/assets/31732943/d997c815-2f95-48c0-856f-688e927718cc)

HTTP/2 is a protocol designed to improve efficiency at the application layer. Unlike its predecessor HTTP/1.1, HTTP/2 introduces *multiplexing*, which allows multiple streams to be sent concurrently over a single TCP connection. This reduces network latency and improves overall performance.

Flow control is essential in this environment to regulate data transfer. Each receiver in an HTTP/2 stream can control how much data it’s willing to accept at a time.

HTTP/2 uses a *credit-based* flow control mechanism. A receiver advertises an initial credit (i.e., how much data it can accept), and this credit is consumed as data arrives. The receiver can replenish credits by sending a `WINDOW_UPDATE` frame. This allows the sending rate to be dynamically adjusted.

Importantly, HTTP/2 flow control is *hop-by-hop*, not end-to-end. Each intermediary node (such as a proxy or gateway) independently manages flow control based on its own resource constraints and policies. This enhances both performance and stability across the network.

<img src="https://github.com/taehyeok-jang/taehyeok-jang.github.io_legacy/assets/31732943/43174293-f4ff-49a6-b0da-e7f6eb097e10" alt="http2_flow_control_01"/>

## Reactive Stream

![bp buffer2 v3](https://github.com/taehyeok-jang/taehyeok-jang.github.io_legacy/assets/31732943/95bca026-205c-44bf-a79d-2502c1a86386)

Reactive stream's *back-pressure* is a mechanism where the *subscriber* controls how much data it is willing to accept from the *publisher*. If the subscriber and publisher are on different systems across the network, their processing speeds are likely to differ. If the subscriber processes data slower than the publisher sends it, a backlog will accumulate on the subscriber side. This can lead to buffer overflows and, in extreme cases, hardware failure.

To prevent this, the subscriber controls the *acceptance rate* by requesting how many data it’s ready to receive via callbacks.

Various implementations of reactive streams support different back-pressure strategies. There is no one-size-fits-all solution—the best strategy depends on the network conditions and application-specific behavior.

## References

* **TCP Layer**

  * [TCP: Flow Control and Window Size (YouTube)](https://www.youtube.com/watch?app=desktop&v=4l2_BCr-bhw)

* **HTTP/2**
  * [LiveBook: HTTP/2 in Action – Chapter 7](https://livebook.manning.com/book/http2-in-action/chapter-7/61)
  * [RFC 7540 – HTTP/2 Specification](https://datatracker.ietf.org/doc/html/rfc7540)
  * [Slideshare: HTTP/2 for Video Streaming](https://www.slideshare.net/Enbac29/http2-standard-for-video-streaming)
  * [Google Web Fundamentals – HTTP/2](https://developers.google.com/web/fundamentals/performance/http2)
  * [HTTP/2 Spec](https://http2.github.io/http2-spec/)

* **Reactive Stream**
  * [RxJava Wiki – Backpressure](https://github.com/ReactiveX/RxJava/wiki/Backpressure)
  * [RxJava 2.x BackpressureStrategy Documentation](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/BackpressureStrategy.html)


