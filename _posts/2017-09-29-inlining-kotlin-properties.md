---
layout: post
title:  "Inlining Kotlin Properties"
date:   2017-09-29 00:00:00 -0400
categories: kotlin
tags: [kotlin]
---
Did you know that you can [inline properties in Kotlin][inline-properties]? I discovered this small 
feature accidentally while browsing through Kotlin docs, and thought it's worth a short article. So 
why is this useful?

Let's take a trivial Android example:

```kotlin
val View.isVisible
  get() = visibility == View.VISIBLE
```

That's a simple yet convenient helper that hides the comparison operation behind a `Boolean` 
property, making your code look like this:

```kotlin
if (button.isVisible) {
  Toast.makeText(context, "I'm a button!", Toast.LENGTH_SHORT).show()
}
```

`isVisible` is actually an extension property, since it's declared on an existing `View` class. 
Extension properties, similar to extension functions, are just syntactic sugar on top of good old 
Java static methods, with the first method parameter becoming the receiver in the extension 
function. If we decompile the previous snippet, using the "Show Kotlin Bytecode" feature (if you're 
in Android Studio, open Tools -> Kotlin -> Show Kotlin Bytecode), we'll see the following Java code:

```java
if(ViewKt.isVisible((View)button)) {
  Toast.makeText(CheckNowView.this.getContext(), (CharSequence)"I'm a button!", 0).show();
}
```

As expected, the code invokes static `isVisible()` method in `ViewKt` class (the file with the 
extension property is called `view.kt`), passing `button` as the first argument. Let's now mark our 
property's getter with `inline` and see what it changes:

```kotlin
val View.isVisible
  inline get() = visibility == View.VISIBLE
```

Decompiled:

```java
View $receiver$iv = (View)button;
if($receiver$iv.getVisibility() == 0) {
  Toast.makeText(CheckNowView.this.getContext(), (CharSequence)"I'm a button!", 0).show();
}
```

As you see, the comparison operation has been inlined on the call site, removing the static method 
invocation. This code looks almost as if you'd write it in Java, but the Kotlin counterpart looks a 
lot neater.

## Declaring Inline Properties

Here are a few notes on the usage of inline properties: with `var`s, you can mark both the getter 
and the setter `inline`:

```kotlin
var View.someProperty: String
  inline get() = "Random value"
  inline set(value) {
    println("Value was set")
  }
```

If this is the case, you can actually mark the property itself `inline`, that will propagate the 
setting to both accessors:

```kotlin
inline var View.someProperty: String
  /* inline */ get() = "Random value"
  /* inline */ set(value) {
    println("Value was set")
  }
```

`inline` is forbidden on properties that have a backing field:

```kotlin
// won't compile!
var upperCaseString: String = ""
  inline get() = field.toUpperCase()
```

The rule also applies to the accessor that doesn't reference the backing field:

```kotlin
// won't compile!
var upperCaseString: String = ""
  get() = field.toUpperCase()
  inline set(value) {
    println("Field set!")
  }
```

And last but not least, keep in mind that inlining will often increase the overall bytecode size, so 
avoid inlining accessors that contain big chunks of code:

```kotlin
// not the best idea!
var aProperty: String
  get() = ""
  inline set(value) {
    // a lot of code here!
  }
```

## Conclusion

When used properly, inlining the properties can help the compiler produce more optimal bytecode, 
removing unnecessary indirections. The trivial example we looked at might give you an idea about 
which accessors are worth inlining. In any case, rely on the decompiler to see exactly what the Java 
counterpart of your code looks like.

Cheers!

[inline-properties]: https://kotlinlang.org/docs/reference/inline-functions.html#inline-properties
