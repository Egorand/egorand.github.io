---
layout: post
title:  "Ok Multiplatform! (droidcon NYC)"
date:   2018-08-27 00:00:00 -0400
tags: [kotlin, open-source, presentation]
---
Co-presented with [Jesse Wilson][jesse].

Okio is a small library that powers a lot of Square’s open source software, such as OkHttp, Moshi 
and Wire. Okio makes I/O easy by solving the most common problems in a simple and efficient way.

At Square, we’re investing in Kotlin. We love the language and the tooling, and we love how Kotlin 
makes us more productive. We’re excited about being able to run Kotlin on multiple platforms, and 
we’d love to be able to harness the power of Okio on Web and iOS - that’s why we’ve embarked on a 
journey to migrate Okio to multiplatform Kotlin!

In this talk we’ll share our experiences and namely:

- What worked for us and what didn’t
- Our strategy for moving fast without breaking code
- Maintaining compatibility: Java source vs Kotlin source vs bytecode
- Issues we’ve encountered along the way and ways to work around them
- Performance considerations
- How this impacts OkHttp, Retrofit, Moshi & Wire

This talk should be of interest to anyone who works with multiplatform Kotlin or wants to learn more
about it.

## Video

<iframe 
  width="560" 
  height="315" 
  src="https://www.youtube.com/embed/Q8B4eDirgk0" 
  title="YouTube video player" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
  allowfullscreen>
</iframe>

[jesse]: https://twitter.com/jessewilson
