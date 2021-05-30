---
layout: post
title:  "Taming the Kotlin Compiler (droidcon Toronto)"
date:   2019-11-13 00:00:00 -0400
tags: [kotlin, presentation]
---
Co-presented with [John Rodriguez][john].

As more and more Android code is being written in Kotlin, and many developers are switching over 
from Java, the performance of the Kotlin compiler has become a hot discussion topic. Javac has been 
tuned and optimized for many years, so how does kotlinc compare?

In addition, there are a whole new family of tools around kotlinc, such as kapt, the Kotin Gradle 
plugin and more. How can one use them efficiently to get optimal performance?

In this talk, we’ll take a deep dive into the following:

* Is Kotlin slower than Java? Should you worry?
* Caching and incrementality of Kotlin compilation
* How to use Kotlin with Gradle efficiently
* What is compilation avoidance and what are the state of things in Kotlin?
* Kotlin compilation in the context of Android

This talk will help you develop a better understanding of how your code is built, what performance 
bottlenecks exist in the toolchain, and how to avoid them to improve your build times.

## Video

<iframe 
  width="560" 
  height="315" 
  src="https://www.youtube.com/embed/EA1Zhf5qUhU" 
  title="YouTube video player" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
  allowfullscreen>
</iframe>

## Slides

<script async 
  class="speakerdeck-embed" 
  data-id="dfde43f49d64413ea2609e0b507e86ea" 
  data-ratio="1.77777777777778" 
  src="//speakerdeck.com/assets/embed.js">
</script>

[john]: https://twitter.com/jrodbx
