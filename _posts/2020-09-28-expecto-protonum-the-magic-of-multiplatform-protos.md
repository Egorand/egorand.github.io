---
layout: post
title:  "Expecto Protonum!: The Magic of Multiplatform Protos (Chicago Roboto)"
date:   2020-09-28 00:00:00 -0400
tags: [android, kotlin, open-source, presentation]
---
Kotlin Multiplatform doesn’t need a formal introduction – while still considered experimental, it’s 
already helping teams around the world to share their business logic across multiple platforms. It’s
like magic!

At Cash App we use Protocol Buffers to model the domain and establish communication between our 
server and client apps. We’re actively working on Wire – a library built at Square that parses proto
files and generates platform-agnostic Kotlin code. Recently we’ve started experimenting with running
generated code on non-JVM platforms, in hopes to someday share even more code between our apps – and
so far it’s been a success!

In this talk, we’ll look at a demo that includes server and client apps that communicate using 
Protocol Buffers. You’ll learn more about how to write and consume Kotlin Multiplatform code on 
Android, server, and iOS, discover some proper use-cases and gotchas, and hopefully get all hyped up
about this exciting technology!

## Video

<iframe 
  src="https://player.vimeo.com/video/466242120" 
  width="640" 
  height="360" 
  frameborder="0" 
  allow="autoplay; fullscreen; picture-in-picture" 
  allowfullscreen>
</iframe>
