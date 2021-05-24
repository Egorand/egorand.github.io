---
layout: post
title:  "How to Try the New Android Studio and Get Away with It"
date:   2017-03-21 00:00:00 -0400
categories: gradle
---
It's that time again when the fresh version of Android Studio hits the Canary channel, and of 
course, you're eager to try it. As soon as you open your project in the new Android Studio, it will 
nicely ask you to update the Android Gradle plugin version to match the IDE version. Depending on 
how adventurous your team is, `alpha` or `beta` plugin versions may or may not be OK. Sometimes your 
colleagues will complain that a preview version of the Android Gradle plugin breaks their build. So 
how can you try an early preview on your development machine without accidentally pushing your 
changes to the shared repo, and without the need to edit the `build.gradle` before every push? 
There's a pretty easy way to achieve this.

First of all, open the `gradle.properties` file in the root directory of your project (create the 
file if it doesn't exist yet). Add the following line:

```groovy
gradlePluginVersion=2.3.0
```

Now navigate to the `build.gradle` file in the root folder of your project and replace this:

```groovy
buildscript {
  dependencies {
    classpath 'com.android.tools.build:gradle:2.3.0'
  }
}
```

with this:

```groovy
buildscript {
  dependencies {
    classpath "com.android.tools.build:gradle:$gradlePluginVersion"
  }
}
```

Build the project to check that this works - it should. You can safely push the changes to the repo. 
This change itself doesn't add or remove anything from the project, but it introduces an indirection 
that you can leverage to substitute the value with a local one. Now, open the
`gradle.properties` file that's located inside the `.gradle` folder in the Home directory of your 
computer (on Mac that's `/Users/you/.gradle/gradle.properties`), and add the following line:

```groovy
gradlePluginVersion=2.4.0-alpha1
```

Get back to Android Studio and build again, you'll notice that the newer version of the plugin got 
picked up. How does this work?

There's a number of ways to parameterize a Gradle build:

- Specifying parameters in the project's `gradle.properties`
- Specifying parameters in computer's `gradle.properties`
- Passing parameters from the command line

The latter is also a working solution, here's how the build command would look:

```groovy
./gradlew -PgradlePluginVersion=2.4.0-alpha1 assemble
```

In the list above, the latest wins, i.e. if you pass a parameter from the command line, it will 
overwrite the parameter with the same key, specified in any of the `gradle.properties` files. And 
hence, your computer's `gradle.properties` wins against the project's one. Take advantage of this 
fact to overwrite project properties without changing them in the project itself. Enjoy!
