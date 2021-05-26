---
layout: post
title:  "Preconditions.kt: Validate Your Kotlin"
date:   2017-08-19 00:00:00 -0400
categories: kotlin
---
Input validation is important, so you're most likely familiar with the following pattern in Java:

```java
public static <T> T getFifthElement(@Nonnull T[] array) {
  if (array == null || array.length < 5) {
    throw new IllegalArgumentException(
        "array should be non-null and have at least five elements!");
  }
  return array[4];
}
```

What would happen if we omitted all that verbose checking and went with a one-liner? The code would 
still break as expected, throwing either a `NullPointerException` or an 
`ArrayIndexOutOfBoundsException`. However, `IllegalArgumentException` clearly indicates that the 
calling code passes invalid arguments, making it a lot easier for the developer to identify the 
problem. Also note, that although we're annotating the parameter with `@Nonnull`, we still have to 
null-check: besides an IDE warning, nothing is preventing the calling code from actually passing in 
a `null` - you love Kotlin, don't you?

To make input validation a little less verbose, Kotlin provides a set of utility functions, 
contained in a file called [`Preconditions.kt`][preconditions-kt]. Most of the functions in this file 
have names starting with either `require` or `check`, let's dive in and see how they work.

## require()

What `require()` does is pretty straightforward:

```kotlin
public inline fun require(value: Boolean, lazyMessage: () -> Any): Unit {
  if (!value) {
    val message = lazyMessage()
    throw IllegalArgumentException(message.toString())
  }
}
```

It takes a `value`, that should resolve to `Boolean`, and a `lazyMessage` function that should 
return the error message. Why a function and not a plain `String`? It's not uncommon to do string 
concatenation to construct the error message, for example, including the argument into the message 
to make it more informative. When wrapped into a function, the concatenation (which can be 
expensive) will only happen when the precondition fails, and not at the time of the function call - 
that's a very nice optimization!

Let's rewrite our Java function in Kotlin using `require()`:

```kotlin
fun <T> getFifthElement(array: Array<T>): T {
  require(array.size >= 5) {
    "array should have at least five elements, but has ${array.size}!"
  }
  return array[4]
}
```

Let's see what we have here:

- First, note that the null-check is gone - compiler handles it for us!
- The precondition has been flipped from `array.length < 5` to `array.size >= 5` and now reads more 
  naturally: "function requires that the size of the array is greater or equal to 5".
- The string creation, which now includes the size of the offending array, will only happen if the 
  precondition fails, so we don't have to worry about performance in the happy case. Looks good!

There's a shorter version of `require()`:

```kotlin
public inline fun require(value: Boolean): Unit =
    require(value) { "Failed requirement." }
```

In my opinion, "Failed requirement" is not very suggestive as the error message, so I would 
recommend using the two-parameter version.

## check()

`check()` is almost a carbon copy of `require()`:

```kotlin
public inline fun check(value: Boolean, lazyMessage: () -> Any): Unit {
  if (!value) {
    val message = lazyMessage()
    throw IllegalStateException(message.toString())
  }
}
```

The only difference is that `check()` throws `IllegalStateException` when the precondition fails. 
The difference is in the semantics:

- `IllegalArgumentException` should be thrown when a method is called with an incorrect set of 
  arguments.
- `IllegalStateException` should be thrown when a method is called at an inappropriate time, i.e. 
  the current state of the object does not allow that particular method to be called.

If this sounds a bit vague, the following snippet should help drive the point home. The code comes 
from [KotlinPoet][kotlinpoet], a handy library that helps you generate Kotlin code.

```kotlin
class TypeSpec {
  ...
  class Builder {
    ...
    fun companionObject(companionObject: TypeSpec) = apply {
      check(kind == Kind.CLASS || kind == Kind.INTERFACE) {
        "$kind can't have a companion object"
      }
      require(companionObject.kind == Kind.COMPANION) {
        "expected a companion object class but was $kind "
      }
      this.companionObject = companionObject
    }
  }
}
```

`companionObject()` is a builder method that adds a companion object definition to a `TypeSpec`. As 
you can see, both `require()` and `check()` are in play here:

- `check()` verifies that the `TypeSpec` that's being created is either a class or an interface: 
  adding a companion object to an enum or an annotation wouldn't make sense. This is a
  validation of the state of the current `Builder` object, and not of the argument of the method, 
  hence the use of `check()`.
- `require()` verifies that `companionObject` actually represents a companion object. This is a 
  validation of the argument, hence the code uses `require()`.

Same as `require()`, `check()` comes in a  shorter form which defaults the error message to "Check 
failed.". Again, I would not recommend using it a lot, as I strongly believe that an exception 
message should be as descriptive as possible to be helpful.

## Conclusion

Kotlin's standard library contains a bunch of hidden gems, that can help you simplify your code and 
make it more idiomatic. `Preconditions.kt` contains a couple of handy functions, `require()` and 
`check()`, that assist in validating input and state of your objects in a nice and
clean way. Sanitize your input, and keep writing Kotlin!

Cheers!

[preconditions-kt]: https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/util/Preconditions.kt
[kotlinpoet]: https://github.com/square/kotlinpoet
