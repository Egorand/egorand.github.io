---
layout: post
title:  "What's the big IDEA? Writing Intellij plugins for Kotlin (KotlinConf)"
date:   2018-10-05 00:00:00 -0400
tags: [intellij, kotlin, presentation]
---
Co-presented with [Alec Strong][alec].

With Kotlin having full interop with Java, mixed codebases have become common and effective - but 
have made writing developer tools more challenging. How do you support multiple languages with a 
single tool? How do you convert existing plugins from Java to Kotlin and is there a way to avoid 
having to?

This talk covers UAST (Universal Abstract Syntax Tree), an API for working with languages 
generically. With UAST you can write a single tool that will work for both Java and Kotlin - no 
special casing needed. We talk about how to setup a plugin to use UAST and walk through a sample 
that works on mixed codebases.

The talk also dives into the types of problems you can solve by writing an IntelliJ plugin, as well 
as other applications for UAST outside of IntelliJ IDEA.

## Video

<iframe 
  width="560" 
  height="315" 
  src="https://www.youtube.com/embed/j2tvi4GbOr4" 
  title="YouTube video player" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
  allowfullscreen>
</iframe>

[alec]: https://twitter.com/Strongolopolis
