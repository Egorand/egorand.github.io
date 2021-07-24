---
layout: post
title: "Hidden Perks of Unit Tests"
date: 2021-07-24 00:09:00 -0400
tags: [testing]
---
Unit testing is one of my favorite software engineering techniques. In our industry, the word 
"testing" is strongly associated with the word "bug", so the common belief is that unit tests help
find/eradicate/prevent bugs. And while this is not untrue, there's much more that unit testing 
can offer! Here's my top 3 of less obvious perks of unit tests:

## Unit tests as a safety net

This, in my opinion, is the most important reason why we write unit tests.

Every active codebase should be cheap to change. As product requirements evolve, new code gets
written and old code gets deleted (hopefully), refactoring should become the norm and not something 
developers shy away from. 

Imagine you enjoy reading, and you're just starting to build your home library. While you own just a 
few books you can simply stack them on your desk, but as your collection grows you should at least
consider splitting the stack into two or more smaller stacks to prevent it from toppling over. Once 
you've bought enough books you should probably consider investing in a bookshelf, and later on - a 
bookcase. Refactoring your personal library when the need arises keeps it tidy and ensures it won't
collapse when you add a new book or grab one to read.

Same thing with code, only that there's likely more risk of breaking things when refactoring. This
makes developers reluctant to change the existing structures and instead keep piling changes on top.
Eventually the code becomes hard to understand and very expensive to change.

Unit tests address this problem by helping you build safety nets around units of code. Executing a
suite of unit tests is supposed to be fast, hence you have a quick way to verify that your code is
still correct. This makes changes cheap - if you accidentally break things the test
suite will immediately alert you, so you can backtrack and fix the issue. The effects of having a
short feedback loop between making a change and verifying its correctness are massive, and unit 
tests unlock this superpower.

## Unit tests as an API design tool

Designing great API is more art than science, but one thing is for sure: you won't know what works
and what doesn't until you start using that API. The more users your API has, the more signal you
get, but it's always better to start small and evolve the API in a pragmatic fashion than to
over-engineer from the beginning.

Write a unit test against your new API to see how it feels to interact with: 

- Is it easy enough to bring the API under test? 
- Do the parameters make sense? 
- How easy is it to assert against the return values? 
- How easy is it to handle errors, and so on.

Setting up a test suite usually requires less effort than building a demo project, so it's a great
way to get some early feedback on your API and start iterating on it.

## Unit tests as a documentation mechanism

Good documentation takes a lot of effort and ongoing maintenance. There's a lot of memes about stale
code comments, I like this one ([source][codinghorror-tweet]):

{:refdef: style="text-align: center;"}
![DaTX9R1VQAEWWIK](/assets/DaTX9R1VQAEWWIK.jpg)
{: refdef}

And what's worse, there is no bulletproof way to ensure you don't forget to update the documentation
when the code changes. On the contrary, unit tests _will_ break unless updated to reflect the 
changes in production code.

[KotlinPoet][kotlinpoet] has a very extensive suite of unit and integration tests, which also serves
as the library's cookbook. Finding multiple examples of how to use an API is as easy as doing a 
quick GitHub search, or a Find Usages lookup after opening the library repo in an IDE. The tests are 
guaranteed to reflect the latest state of the API - otherwise the project builds would not pass.

## Conclusion

Unit testing isn't just about debugging, and I would argue that the biggest benefits of unit testing
aren't even related to debugging. First and foremost it's a technique for establishing fast 
iteration loops around units of code, decreasing the cost of changes and making the effects (wanted 
or unwanted) of those changes visible immediately. Quality unit tests unlock the ability to safely 
refactor the codebase as often as needed, which is the key to preventing it from becoming a mess. So 
write unit tests to ship safe code fast!

 [kotlinpoet]: https://github.com/square/kotlinpoet
 [codinghorror-tweet]: https://twitter.com/codinghorror/status/983148100301742080
