---
layout: post
title:  "Why Trailing Commas In Kotlin Are Hype"
date:   2020-03-14 00:00:00 -0400
categories: kotlin
---
Kotlin 1.3.70 landed last week with a bunch of new APIs, bug fixes and tooling improvements (check out the [release blog post][release-blog-post]), but a small change that mostly flew under the radar is this:

- [KT-34743][kt-34743] Support trailing comma in the compiler

The compiler now allows leaving a dangling comma after function, constructor, lambda parameters, and many other places where it was previously forbidden. Why does it matter? The answer is - the diffs! Let's look at an example:

{% highlight kotlin %}
class NoCommas(
  val foo: Int
)

class YesCommas(
  val foo: Int,
)
{% endhighlight %}

We've got two simple classes, each declaring a single property inside the primary constructor. The property declaration in `YesCommas` has a trailing comma. Looks ugly, right? Well, let's see what happens when we add more properties to these classes:

{% highlight diff %}
  class NoCommas(
-    val foo: Int
+    val foo: Int,
+    val bar: Int
  )

  class YesCommas(
    val foo: Int,
+   val bar: Int,
  )
{% endhighlight %}

Look how cleaner the second diff looks. When we added a property to `NoCommas`, we had to add a comma at the end of the existing property declaration, which showed up in the diff as a modified line. This is unfortunate, since our intent was to only *add* a property and make no modifications to existing code. The `YesCommas` example communicates that intent much better: since the existing property declaration already had a trailing comma, we didn't need to modify any lines of code to declare a new property.

### The state of trailing commas in Kotlin

Although, as mentioned above, the compiler accepts trailing commas as of Kotlin 1.3.70, if you simply upgrade your Kotlin dependencies to 1.3.70 and try to use this feature, you'll likely get hit with the following error message:

> The feature "trailing commas" is only available since language version 1.4

Therefore, the only way you can start using trailing commas in your project is by setting the language version to 1.4, which at this point in time is likely too early for production code. But to soften the blow, let me show you a language feature that already supports trailing commas - enums! Here's an example:

{% highlight kotlin %}
enum class Enum {
  FOO,
  BAR;

  override fun toString() = super.toString().toLowerCase()
}
{% endhighlight %}

We'll try to add another entry to this enum:

{% highlight diff %}
  enum class Enum {
    FOO,
-   BAR;
+   BAR,
+   BAZ;

    override fun toString() = super.toString().toLowerCase()
  }
{% endhighlight %}

That's the same problem we've seen earlier, the diff looks messier than it should. Luckily, Kotlin allows us to format enums in the following way:

{% highlight kotlin %}
enum class Enum {
  FOO,
  BAR,
  ;

  override fun toString() = super.toString().toLowerCase()
}
{% endhighlight %}

Which leads to much cleaner diffs when we introduce new enum entries:

{% highlight diff %}
  enum class Enum {
    FOO,
    BAR,
+   BAZ,
    ;

    override fun toString() = super.toString().toLowerCase()
  }
{% endhighlight %}

While cleaner version control diffs might not seem like such a big deal, they definitely help make code reviews more straightforward and pleasant by highlighting the intent of every code change and removing focus from syntactic noise. And that's why trailing commas coming to Kotlin is hype!

[release-blog-post]: https://blog.jetbrains.com/kotlin/2020/03/kotlin-1-3-70-released/
[kt-34743]: https://youtrack.jetbrains.com/issue/KT-34743
