---
layout: post
title:  "A Curious Case of Multiple Locales"
date:   2017-01-28 00:00:00 -0400
categories: android
---
One of the features introduced in Android N is the possibility to select multiple languages in 
Settings ([Language and Locale][multilingual-support]), which should enable the system to resolve 
app locales in a smarter way. Here's how it works.

Imagine a person, whose native language is Italian, but who can speak German as well. The person is 
an owner of an Android smartphone running Android Marshmallow, and the device language is set to 
Italian. Now, the person downloads a simple "Hello World!" app, which supports two locales - English 
(primary) and German. Here's how the app looks on her
device:

{:refdef: style="text-align: center;"}
![b3lZ8p3](/assets/b3lZ8p3.png)
{: refdef}

Since the app does not provide resources in Italian, the system falls back to using the default 
locale, which is English. There's no way for the system to know that our user understands German.

After some time, our user decides to upgrade to a new shiny Pixel smartphone powered by Android 
Nougat. She discovers the multiple language support feature in the Settings and sets up two 
languages, Italian (primary) and German. She installs "Hello World!" app on her new device:

{:refdef: style="text-align: center;"}
![NKQagYg](/assets/NKQagYg.png)
{: refdef}

A lot better! Now the device knows that one of the languages its user speaks is German, and it's 
able to choose German as the app locale.

Now, suppose that we decided to make our simple app backward compatible to support older Android 
versions, and added the `appcompat-v7` library to the Gradle setup. The user downloads the update, 
and what does she see?

{:refdef: style="text-align: center;"}
![f0o8GDR](/assets/f0o8GDR.png)
{: refdef}

Whoops! Looks like our app started showing texts in English, but why? Here's what happened: 
`appcompat-v7` contains resources for the Italian locale, and since all resources are merged at 
build time, our app now contains them too. The system sees the app as supporting user's primary 
locale - which is Italian - and starts using corresponding resources. However, we haven't provided 
an Italian translation for "Hello World!" string, so the system has no other way but to grab the 
string for the default locale, which is English! What do we do now?

Fortunately, there's an easy solution provided by the Android Gradle plugin - the 
[`resConfigs`][res-configs] method. This method allows us to pass in resource qualifiers, and the 
build will strip out resources in all folders that don't match these qualifiers. Here's what we have 
to add to our `app/build.gradle` file:

```groovy
defaultConfig {
  ...

  resConfigs "en", "de"
}
```

This tells Gradle that we're only supporting two locales, `en` and `de`, so all other resources 
won't be bundled into the APK.

As **Amal Samally** pointed out in the comments, a better way to achieve the same effect would be 
the following:

```groovy
defaultConfig {
  ...

  resConfigs "auto"
}
```

Advantage of this method is that you won't need to update the configuration after adding 
translations to your application.

An easy way to verify that your build packages correct resource configurations is to analyze the 
APK. In Android Studio, navigate to Build -> Analyze APK... and select your app's APK in the build
folder. Here's what we could see before adding `resConfigs`:

![08m49ap](/assets/08m49ap.png)

You can see that there are 82 resource configurations in total. After we add `resConfigs` and 
rebuild the APK, we see the following:

![T0qwHfX](/assets/T0qwHfX.png)

Now there are just 2 configurations, the ones we've explicitly specified.

### Conclusion

Support for multiple languages is a great feature and it will certainly improve the experience for 
bilingual users. However, if you're not careful with resource configurations that your app 
packages - and those are not only the ones your app provides but all resources coming from all AAR 
libraries you use - your users won't be able to benefit from improved locale resolution. If your app 
uses `appcompat-v7`, consider using `resConfigs` to work around this problem.

[multilingual-support]: https://developer.android.com/guide/topics/resources/multilingual-support.html
[res-configs]: https://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.ProductFlavor.html#com.android.build.gradle.internal.dsl.ProductFlavor:resConfigs(java.lang.String%5B%5D)
