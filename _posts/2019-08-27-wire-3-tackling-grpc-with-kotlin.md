---
layout: post
title:  "Wire 3: Tackling gRPC with Kotlin (droidcon NYC)"
date:   2019-08-27 00:00:00 -0400
tags: [android, kotlin, open-source, presentation]
---
Co-presented with [Benoît Quenaudon][benoit].

Protocol Buffers (or Protobuf) is an efficient schema-based data serialization protocol, and gRPC is
a high-performance, HTTP/2-based RPC framework. The two work together flawlessly to help you build 
world class distributed systems.

At Square, client and server engineers collaborate on Protobuf schemas to define APIs. We also built
Wire - a library which processes schemas and generates Java code that applications can use to send 
and receive data. Last year we started working on Wire 3, which is rewritten in Kotlin, generates 
Kotlin code, and adds a number of exciting features:

- Protobuf messages as data classes
- Multiplatform runtime module
- Coroutines-based gRPC APIs
- Gradle plugin

In this session, we’ll take a deep dive into these features, and talk about how we leveraged Kotlin 
to create better APIs. We’ll show you how to get the best out of Protobuf and gRPC for your server 
and client applications using Wire.

## Video

<iframe 
  title="vimeo-player" 
  src="https://player.vimeo.com/video/362742306" 
  width="640" 
  height="360" 
  frameborder="0" 
  allowfullscreen>
</iframe>

## Slides

<script async 
  class="speakerdeck-embed" 
  data-id="64c83e6ced764d32b41371ff6f558939" 
  data-ratio="1.77777777777778" 
  src="//speakerdeck.com/assets/embed.js">
</script>

[benoit]: https://twitter.com/oldergod
