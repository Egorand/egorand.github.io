---
layout: post
title:  "State of Android Testing in 2022 (droidcon Italy)"
date:   2022-10-07 00:00:00 +0200
tags: [android, kotlin, testing, presentation]
---
Android’s testing story has come a long way since the early days (old timers might recall not being able to run unit 
tests on the local JVM, or struggling to write UI tests in the low on caffeine pre-Espresso world), and nowadays 
there’s a multitude of tools, both 1st and 3rd party, to help you assemble a world-class testing pipeline. In this talk 
we’ll go down the test pyramid and present an overview of tools and techniques to help you write efficient tests in 
each category. We’ll discuss technologies that are heavily used at (and some even built by) Cash App. The set of topics 
includes, but is not limited to:

- What kinds of tests should you focus on?
- Writing end-to-end tests with Espresso and running them on real devices in the Firebase Test Lab.
- Using Paparazzi to write screenshot tests and catch UI regressions.
- Turbine & other useful tools for testing Kotlin’s coroutines & Flows.

This talk should give developers a holistic overview and a wealth of practical advice on writing all types of tests for 
your Android app!

## Slides

<script async 
  class="speakerdeck-embed" 
  data-id="3050a667dd2941578e75f998d66dc617" 
  data-ratio="1.77777777777778" 
  src="//speakerdeck.com/assets/embed.js">
</script>
