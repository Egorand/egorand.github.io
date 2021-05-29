---
layout: post
title:  "\"Your Espresso, Sir!\", or Stepping Up Your UI Tests With Test Butler"
date:   2016-08-11 00:00:00 -0400
tags: [android, kotlin, testing]
---
**Fact #1. Acceptance tests are great.** Continuous integration is slowly becoming ubiquitous in 
mobile app development, and having a Jenkins instance run your entire regression test suite during 
the night, when everybody on the team is dreaming of sweet Marshmallows and Nougat, or even multiple 
times per day, is a great addition to a successful delivery process.

**Fact #2. Acceptance tests are hard.** Automating a test on Android is not an easy task. In recent 
years, frameworks like [Espresso][espresso] have made the task a lot easier, taking care of the 
details, but still - there are so many things that can go wrong during a UI test! It's actually a 
good thing when an acceptance test fails due to fair reasons - it helps you locate a bug before your 
app goes out to your users, but it's extremely frustrating when your tests fail for whatever 
different reasons - network is down, keyguard gets in the way, or your fancy animations confuse the 
test runner. Flaky tests would eventually make developers trust them less and run them less, meaning 
that the effort that went into setting up the test infrastructure goes out the window.

Turns out the Android team at LinkedIn faced this kind of problems with their UI tests, and they 
came up with a pretty nice solution - a library called [Test Butler][test-butler-article]. Test 
Butler is a two-part project, consisting of an app, signed with the system keystore, which grants it 
access to many hidden emulator settings, and a small library, that talks to the app through an AIDL 
interface. Now that Test Butler is on [GitHub][test-butler-github], let's set it up and see what it 
can help us with!

### Setting up Test Butler

### Installing the helper app

First, you'll need the Test Butler helper app installed on your emulator. Currently, there's no 
easier way than downloading the APK from [Bintray](https://bintray.com/drewhannay/maven/test-butler-app) and installing it manually. Alternatively, you can clone the project from GitHub and build the app yourself, the debug build type is configured to sign the APK with system keystore. In future there could probably be a Gradle plugin that would download and install the helper app automatically before the tests run.

### Adding the Test Butler library to your project

The library part is available from Bintray, simply add a dependency to the `build.gradle` file:

```groovy
androidTestCompile 'com.linkedin.testbutler:test-butler-library:1.0.0'
```

### Create a custom test runner

We'll need a custom test runner class that will establish the communication between our tests and the helper app, here's how it should look:

```kotlin
class TestButlerTestRunner : AndroidJUnitRunner() {

    override fun onStart() {
        TestButler.setup(InstrumentationRegistry.getTargetContext())
        super.onStart()
    }

    override fun finish(resultCode: Int, results: Bundle?) {
        TestButler.teardown(InstrumentationRegistry.getTargetContext())
        super.finish(resultCode, results)
    }
}
```

The sample project used for this article is available on [GitHub][test-butler-app]. The code is 
written in Kotlin, hope it's straightforward to read even if you're not familiar with the language.

Don't forget to reference the test runner class inside `build.gradle` file:

```groovy
defaultConfig {
    // other config

    testInstrumentationRunner "me.egorand.testbutlerdemo.TestButlerTestRunner"
}
```

Now we're all set, let's write some tests that exercise a feature in Test Butler that I find 
particularly useful - screen rotation.

### Testing Screen Rotation

Let's test the following scenario: we have alternative layouts for portrait and landscape versions 
of our `MainActivity` and we'd like to check that the correct layouts are used in both use cases. 
The `layout/activity_main.xml` contains a single `Fragment` called `MasterFragment`:

```xml
<fragment
    android:id="@+id/master_fragment"
    class="me.egorand.testbutlerdemo.MasterFragment"
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="me.egorand.testbutlerdemo.MainActivity"
    tools:layout="@layout/layout_fragment"/>
```

and `layout-land/activity_main.xml` combines `MasterFragment` and `DetailFragment` in a single 
layout:

```xml
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal">
    <fragment
        android:id="@+id/master_fragment"
        class="me.egorand.testbutlerdemo.MasterFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1"
        tools:layout="@layout/layout_fragment"/>
    <fragment
        android:id="@+id/detail_fragment"
        class="me.egorand.testbutlerdemo.DetailFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1"
        tools:layout="@layout/layout_fragment"/>
</LinearLayout>
```

Normally it would be challenging to implement this test scenario, to rotate a device in test
mode. You can definitely setup a dedicated emulator that is turned into landscape mode, but with 
Test Butler you could just reuse a single emulator for both test cases:

```kotlin
class OrientationChangeTest {

    @Rule @JvmField val activityTestRule = ActivityTestRule<MainActivity>(MainActivity::class.java)

    @Test fun shouldDisplayMasterFragmentInPortrait() {
        rotateToPortrait()

        onView(withText(R.string.master_message)).check(matches(isDisplayed()))
    }

    @Test fun shouldDisplayMasterAndDetailFragmentsInLandscape() {
        rotateToLandscape()

        onView(withText(R.string.master_message)).check(matches(isDisplayed()))
        onView(withText(R.string.detail_message)).check(matches(isDisplayed()))
    }
}
```

`rotateToPortrait()` and `rotateToLandscape()` are simply wrapper functions for Test Butler's APIs:

```kotlin
fun rotateToPortrait() {
    TestButler.setRotation(Surface.ROTATION_0)
    Thread.sleep(SLEEP_VALUE)
}

fun rotateToLandscape() {
    TestButler.setRotation(Surface.ROTATION_90)
    Thread.sleep(SLEEP_VALUE)
}
```

Unfortunately, at the moment we'll need to `sleep()` for a second or two to make sure the rotation 
is applied, otherwise the tests can fail randomly.

Neat! But as I said, keeping an emulator in landscape mode around is usually not a problem, so 
what's the gain? Let's now try a different scenario, which is really hard to achieve without the 
Test Butler magic.

### Testing Instance State Persistence

To ensure proper UX on orientation change you should save and restore the instance state of your 
`Activity`s and `Fragment`s. This can be a source of subtle bugs, so having automated tests that 
verify this behavior can be quite valuable. Let's try to test this behavior inside `InputActivity`, 
that has a single `EditText` in the layout. `InputActivity` must persist the text entered into the 
`EditText` between orientation changes. Here's how it looks:

```kotlin
class InputActivity : AppCompatActivity() {

    companion object {
        val STATE_INPUT = "state_input"
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_input)

        if (savedInstanceState != null) {
            input.setText(savedInstanceState.getString(STATE_INPUT))
        }
    }

    override fun onSaveInstanceState(outState: Bundle?) {
        super.onSaveInstanceState(outState)
        outState?.putString(STATE_INPUT, input.text.toString())
    }
}
```

Now let's write a test that verifies this behavior:

```kotlin
class SavingInstanceStateTest {

    @Rule @JvmField val activityTestRule = ActivityTestRule<InputActivity>(InputActivity::class.java)

    @Test fun shouldSaveAndRestoreInstanceState() {
        val text = "Hello world!"
        onView(withId(R.id.input)).perform(typeText(text))

        rotateToLandscape()

        onView(withId(R.id.input)).check(matches(withText(text)))
    }
}
```

As you see, we're typing the text into the `EditText`, rotating the device into landscape and 
checking whether the text has been restored. I'd say that's quite a good test to have in your 
regression suite!

Device rotation is just one of a bunch of useful features provided by Test Butler. I encourage you 
to read Drew Hannay's [Open Sourcing Test Butler][test-butler-article] article, and check out the 
[GitHub project repo][test-butler-github].

Happy testing!

[espresso]: https://google.github.io/android-testing-support-library/docs/espresso/
[test-butler-article]: https://engineering.linkedin.com/blog/2016/08/introducing-and-open-sourcing-test-butler--reliable-android-test
[test-butler-github]: https://github.com/linkedin/test-butler
[test-butler-app]: https://bintray.com/drewhannay/maven/test-butler-app
