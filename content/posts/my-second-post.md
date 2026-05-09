+++
date = '2026-05-09T12:20:40-04:00'
draft = true
title = 'My Second Post'
+++

Voice AI only feels natural if conversation moves at the speed of speech. When the network gets in the way, people hear it immediately as awkward pauses, clipped interruptions, or delayed barge-in. That matters for ChatGPT voice, for developers building with the Realtime API, for agents working in interactive workflows, and for models that need to process audio while a user is still talking.

At OpenAI’s scale, that translates into three concrete requirements:

Global reach for more than 900 million weekly active users
Fast connection setup so a user can start speaking as soon as a session begins
Low and stable media round-trip time, with low jitter and packet loss, so turn-taking feels crisp
The team at OpenAI responsible for real-time AI interactions recently rearchitected our WebRTC stack to address three constraints that started to collide at scale: one-port-per-session media termination does not fit OpenAI infrastructure well, stateful ICE (Interactive Connectivity Establishment) and DTLS (Datagram Transport Layer Security) sessions need stable ownership, and global routing has to keep first-hop latency low. In this post, we walk through the split relay plus transceiver architecture we built to preserve standard WebRTC behavior for clients while changing how packets are routed inside OpenAI’s infrastructure.

WebRTC lets us make real-time AI products
WebRTC is an open standard for sending low-latency audio, video, and data between browsers, mobile apps, and servers. It’s often associated with peer-to-peer calling, but it’s also a practical foundation for client-to-server real-time systems because it standardizes the hard parts of interactive media: ICE for connectivity establishment and NAT (Network Address Translation) traversal, DTLS and SRTP (Secure Real-time Transport Protocol) for encrypted transport, codec negotiation for compressing and decoding audio, RTCP (Real-time Transport Control Protocol) for quality control, and client-side features such as echo cancellation and jitter buffering.

That standardization matters for AI products. Without WebRTC, every client would need a different answer for how to establish connectivity across NATs, encrypt media, negotiate codecs (the coder-decoders selected for transmission and decompression) and adapt to changing network conditions. With WebRTC, we can build on a protocol stack that’s already implemented across browsers and mobile platforms, focusing our own work on the infrastructure that connects real-time media to models.

We also build on the WebRTC ecosystem itself, including mature open-source implementations and the standard work that keeps browsers, mobile apps, and servers interoperable. Foundational work by Justin Uberti (one of WebRTC’s original architects) and Sean DuBois (creator and maintainer of Pion) made it possible for teams like ours to build on battle-tested media infrastructure rather than reinvent low-level transport, encryption, and congestion-control behavior. We’re fortunate that both Justin and Sean are now colleagues here at OpenAI, helping guide how we bring WebRTC and real-time AI closer together.

For AI, the most important property is that audio arrives as a continuous stream. A spoken agent can begin transcribing, reasoning, calling tools, or generating speech while the user is still talking, instead of waiting for a full upload. That’s the difference between a system that feels conversational and one that feels like push-to-talk.

Choosing a media architecture
Once we chose WebRTC, the next question was where to terminate it (where we’d accept and own the WebRTC connection—for example, at the edge) and how to connect those sessions to the inference backend. Termination matters because it determines how we handle real-time session state, media transport, routing, latency, and failure isolation.