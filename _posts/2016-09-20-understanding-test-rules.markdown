---
layout: post
title:  "Understanding Test Rules"
date:   2016-09-20 00:00:00 -0400
categories: java
---
If you've ever happened to write Espresso tests, you should be familiar with the following 
declaration:

```java
@Rule public ActivityTestRule<MainActivity> activityRule =
        new ActivityTestRule<>(MainActivity.class);
```

The addition of `@Rule`s is a relatively new thing in Espresso, historically you would inherit from 
[`ActivityInstrumentationTestCase2`][activity-instrumentation-test-case-2] (deprecated in API level 
24) and use its `getActivity()` method to launch the `Activity` before test. As in many cases, 
inheritance turned out to be a limiting solution - what if you want to introduce your own base class 
containing common logic, which is not necessarily an `ActivityInstrumentationTestCase2`? JUnit4's 
rules are a pretty good replacement, here's what the [Rules][rules] wiki says:

> Rules allow very flexible addition or redefinition of the behavior of each test method in a test 
> class.

Indeed, with rules you can plug in desired behavior, instead of inheriting it from a single base 
class. Additionally, it's pretty easy to write custom rules. In this article, we'll dive deeper into 
how rules work, and will write a custom rule that will allow us to manage a `Realm` instance while 
testing an on-disk repository.

### So how does it work?

There's a great article called [Using Rules To Influence JUnit Test Execution][junit-rules], which 
provides a pretty detailed explanation of how rules work and how JUnit handles them, I'll just 
mention the important bits:

There are 3 parts of the equation:

- `Statement` classes, which augment test execution and can run custom code before and/or after the 
  actual test.
- Rule classes, which decide which `Statement`s to use.
- `@Rule` annotations, which tell JUnit which rules to apply to your test methods.

The process works roughly like this:

- JUnit scans your test class to find all fields annotated with `@Rule`.
- JUnit calls each rule's `apply()` method, passing information about the test method it's currently 
  running, along with `Statement`s it already gathered from previous rules.
- The rule class creates an instance of `Statement` and returns it from `apply()`.
- Eventually, `evaluate()` method should be called on the `Statement`, created by the last rule, in 
  order to trigger test execution.

Let's introduce an example that illustrates these concepts.

### Writing a Custom Test Rule

We've got a class called `PersonsDiskRepo`, it helps us serialize `Person` objects to disk and 
execute different queries on the data. The draft of the class looks like this:

```kotlin
class PersonsDiskRepo(private val realm: Realm) {

    fun getAllPersons(): List<Person> =
            realm.where(Person::class.java)
                    .findAll()

    fun getAllPersonsOlderThan(age: Int): List<Person> =
            realm.where(Person::class.java)
                    .greaterThan(Person.FIELD_AGE, age)
                    .findAll()
}
```

We're using [Realm][realm] for writing data to disk. Now we'd like to write a couple of 
instrumentation tests for this functionality, to verify that the data gets queried properly on the
actual device. Here's our test class:

```kotlin
class PersonsDiskRepoTest {

    companion object {
        val TEST_PERSONS = listOf(
                Person("Alice", 26),
                Person("Bob", 55),
                Person("Christie", 18))
    }

    lateinit var realm: Realm
    lateinit var repo: PersonsDiskRepo

    @Before fun setUp() {
        val config = RealmConfiguration.Builder(InstrumentationRegistry.getTargetContext()).build()
        Realm.setDefaultConfiguration(config)
        realm = Realm.getDefaultInstance()
        realm.executeTransaction {
            TEST_PERSONS.forEach { realm.insert(it) }
        }
        repo = PersonsDiskRepo(realm)
    }

    @Test fun shouldLoadAllPersons() {
        val result = repo.getAllPersons()

        assertThat(result.size).isEqualTo(TEST_PERSONS.size)
        assertThat(result).containsAll(TEST_PERSONS)
    }

    @Test fun shouldLoadPersonsAgedOver20() {
        val result = repo.getAllPersonsOlderThan(20)

        assertThat(result.size).isEqualTo(2)
        assertThat(result).contains(TEST_PERSONS[0])
        assertThat(result).contains(TEST_PERSONS[1])
        assertThat(result).doesNotContain(TEST_PERSONS[2])
    }

    @After fun tearDown() {
        realm.executeTransaction { realm.deleteAll() }
    }
}
```

You can see that our `setUp()` method contains code to configure an instance of `Realm`, and 
`tearDown()` has the code to clean the database before the next test runs. This is something we'd 
like to reuse, as we're likely to have other classes that rely on `Realm`. Let's do it the old way 
and create a base class that will manage `Realm`:

```kotlin
abstract class BaseRealmTest {

    lateinit var realm: Realm

    @Before fun initRealm() {
        val config = RealmConfiguration.Builder(InstrumentationRegistry.getTargetContext()).build()
        Realm.setDefaultConfiguration(config)
        realm = Realm.getDefaultInstance()
    }

    @After fun cleanRealm() {
        realm.executeTransaction { realm.deleteAll() }
    }
}
```

In `PersonsDiskRepoTest`'s `setUp()` we can reference parent's `realm` field to initialize the 
tests:

```kotlin
@Before fun setUp() {
    realm.executeTransaction {
        TEST_PERSONS.forEach { realm.insert(it) }
    }
    repo = PersonsDiskRepo(realm)
}
```

and we can drop `tearDown()` since the cleanup logic will now reside in `BaseRealmTest`. So far so 
good, but what if we had a test class inheriting from the good old 
`ActivityInstrumentationTestCase2` and wanted to use the logic from `BaseRealmTest`? We can't have 
two super classes in Java, right? Enter JUnit rules! Let's now refactor `Realm` management logic 
into a custom rule class and inject it into our `PersonsDiskRepoTest` with the help of the `@Rule` 
annotation.

```kotlin
class RealmRule : TestRule {

    lateinit var realm: Realm

    override fun apply(base: Statement?, description: Description?) = RealmStatement(base)

    inner class RealmStatement(private val base: Statement?) : Statement() {

        override fun evaluate() {

            fun initRealm() {
                val context = InstrumentationRegistry.getTargetContext()
                val config = RealmConfiguration.Builder(context).build()
                Realm.setDefaultConfiguration(config)
                realm = Realm.getDefaultInstance()
            }

            try {
                initRealm()
                base?.evaluate()
            } finally {
                realm.executeTransaction { realm.deleteAll() }
            }
        }
    }
}
```

Let's see what we've got here: our rule class is called `RealmRule` and it implements JUnit's 
`TestRule` interface. There's only one method inside called `apply()`, which JUnit uses to provide 
us with a previously created `Statement` and a `Description` object, containing information about 
the test method JUnit is currently running. Inside `apply()` we're creating an instance of 
`RealmStatement` - a custom subclass of `Statement` that we've created. This class does three 
things:

- Initializes `Realm` and passes the reference to the rule class to make it accessible by test 
  classes.
- Calls `evaluate()` on another `Statement`, which JUnit provided via rule's `apply()` method. This 
  will eventually trigger test execution.
- Cleans up the database inside the `finally` block.

All that's left is to declare an `@Rule`-annotated field inside our test class and assign an 
instance of `RealmRule` to it:

```kotlin
@Rule @JvmField val realmRule = RealmRule()

@Before fun setUp() {
    realmRule.realm.executeTransaction {
        TEST_PERSONS.forEach { realmRule.realm.insert(it) }
    }
    repo = PersonsDiskRepo(realmRule.realm)
}
```

We can reference the instance of `Realm` through `realmRule` inside our `setUp()` method.

Neat! Here's what we achieved:

- We created a reusable rule class that many test classes in our codebase will potentially benefit 
  from.
- We're not forcing our test classes to use inheritance, that can be pretty limiting.
- Our test classes can use `RealmRule` along with any other rules that they require.

Feel free to check out the full source code for this example, which is available on 
[GitHub][github].

### Conclusion

JUnit rules provide a flexible solution for reusing test logic. As we've seen, it's pretty easy to 
write and apply custom test rules. Hope this article will encourage you to refactor your tests and 
get rid of base classes in favor of JUnit rules.

Cheers!

[activity-instrumentation-test-case-2]: https://developer.android.com/reference/android/test/ActivityInstrumentationTestCase2.html
[rules]: https://github.com/junit-team/junit4/wiki/rules
[junit-rules]: http://cwd.dhemery.com/2010/12/junit-rules/
[realm]: https://realm.io/
[github]: https://github.com/Egorand/android-test-rules
