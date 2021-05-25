---
layout: post
title:  "Testing a sorted list with Espresso"
date:   2015-11-21 00:00:00 -0400
categories: android
---
[Espresso][espresso] is a pretty powerful tool when it comes to writing acceptance tests for 
Android. [Acceptance tests][acceptance-tests] are the ones that make sure your entire feature (or a 
certain aspect of the feature) is implemented correctly. The advantage of automating acceptance 
tests is the simplicity of catching regressions, which usually occur during active development or 
bug fixing. When you have automated tests, it all comes down to just running them to find out if you
broke something by a recent change. Awesome!

In this post I'll show you how to setup Espresso in your Android project, and will write a simple 
acceptance test, that checks that a list of Premier League teams is sorted alphabetically. Brew 
yourself a cup of espresso and fasten your seatbelt!

### Espresso setup

It's pretty easy to configure Espresso if you're using Android Studio and Gradle. Just open the 
`build.gradle` file inside the `app` module and add the following dependencies:

```groovy
def APPCOMPAT_VERSION = "24.0.0"
def ESPRESSO_RUNNER_VERSION = "0.5"

dependencies {
    // dependencies with "compile" scope go here

    androidTestCompile "com.android.support:support-annotations:${APPCOMPAT_VERSION}"
    androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2.2'
    androidTestCompile "com.android.support.test:runner:${ESPRESSO_RUNNER_VERSION}"
    androidTestCompile "com.android.support.test:rules:${ESPRESSO_RUNNER_VERSION}"
}
```

By the way, the complete source code for this project is available on [GitHub][github], so feel free
to checkout.

Sync your project with Gradle and sip some coffee while Gradle runs the build. One last detail to 
finalize the setup is to add the following line to `build.gradle` under `defaultConfig` closure:

```groovy
defaultConfig {
    // default setup here

    testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
}
```

And we're done! Let's continue with writing an acceptance test with Espresso.

### Writing an acceptance test

You'll notice the `androidTest` directory under the `src`: this is where the Espresso tests usually 
live. Go ahead and create a class called `TeamsActivityTest`. Add a couple of annotations to make 
your class look like that:

```java
@RunWith(AndroidJUnit4.class)
@LargeTest
public class TeamsActivityTest {
}
```

We declare that we'll be using JUnit version 4 to write our tests. The [@LargeTest][large-test] 
annotation serves as a hint to the test runner regarding what kind of tests our class contains, and 
is usually used with Espresso tests. Next, we'll add the following field to the test class:

```java
@Rule public ActivityTestRule<TeamsActivity> activityTestRule =
        new ActivityTestRule<>(TeamsActivity.class);
```

Until JUnit 4 support was introduced, Android test classes were usually extending the 
[ActivityInstrumentationTestCase2][activity-instrumentation-test-case-2] class. With JUnit 4 it's 
enough to add a field of type [ActivityTestRule][activity-test-rule], annotated with `@Rule`, to 
describe how the `Activity` under test should be started. Check the overloaded versions of 
`ActivityTestRule` constructor to see which tweaks can be made to the test startup.

With our test class set, let's proceed straight to implementing our test case:

```java
@Test
public void teamsListIsSortedAlphabetically() {
    onView(withId(android.R.id.list)).check(matches(isSortedAlphabetically()));
}
```

`onView()`, `withId()` and `matches()` are all static methods inside the framework, so I prefer to 
use static imports to make the test definitions look clean and concise. Check the 
[sample code][github] on GitHub for the correct imports.

`isSortedAlphabetically()` though is a custom [Hamcrest matcher][matcher], that describes the 
condition that we want to check on our `View`, namely, that the contents of the `android.R.id.list` 
are sorted in alphabetical order. Here's how we define the matcher:

```java
private static Matcher<View> isSortedAlphabetically() {
    return new TypeSafeMatcher<View>() {

        private final List<String> teamNames = new ArrayList<>();

        @Override
        protected boolean matchesSafely(View item) {
            RecyclerView recyclerView = (RecyclerView) item;
            TeamsAdapter teamsAdapter = (TeamsAdapter) recyclerView.getAdapter();
            teamNames.clear();
            teamNames.addAll(extractTeamNames(teamsAdapter.getTeams()));
            return Ordering.natural().isOrdered(teamNames);
        }

        private List<String> extractTeamNames(List<Team> teams) {
            List<String> teamNames = new ArrayList<>();
            for (Team team : teams) {
                teamNames.add(team.name);
            }
            return teamNames;
        }

        @Override
        public void describeTo(Description description) {
            description.appendText("has items sorted alphabetically: " + teamNames);
        }
    };
}
```

We know for sure that we're using a `RecyclerView`, so we can safely cast the parameter of 
`matchesSafely()` and extract the `TeamsAdapter` to get to the data. We'll `extractNames()` from the
list of `Team` objects, and then use [Guava's][guava] [Ordering][ordering] class to check that the 
list is properly sorted. When writing Hamcrest matchers, don't overlook the `describeTo()` method, 
as it can prove very useful when your test fails. In our `describeTo()` we'll shortly describe what 
the matcher is doing and will also print the data that we stored locally: now when our test fails 
we'll know exactly how the dataset looked and can draw conclusions on why the condition has failed.

Now you may wonder where do things like `Team` and `TeamAdapter` (or even `RecyclerView` that we 
haven't integrated yet) come from. Writing tests that don't even compile is actually pretty fine 
with [Test Driven Development][tdd], that introduces the "red-green-refactor" cycle: write your 
tests, make them compile and pass, refactor to remove duplication. We're currently "red", so let's 
work our way to "green" by writing some code.

First, integrate the `RecyclerView` by adding the following dependency to the `app/build.gradle` 
file:

```groovy
dependencies {
    // other "compile" dependencies go here
    compile "com.android.support:recyclerview-v7:${APPCOMPAT_VERSION}"

    // "androidTest" dependencies are here
}
```

In case you already have a `MainActivity` in your project, rename it to `TeamsActivity`, otherwise 
create it from scratch. `TeamsActivity` will use the following [layout][activity-main]. `Team` is 
our POJO and it looks like this:

```java
public class Team {

    public final String name;
    public final @DrawableRes int logoRes;

    public Team(@NonNull String name, @DrawableRes int logoRes) {
        this.name = name;
        this.logoRes = logoRes;
    }

    public static final Comparator<Team> BY_NAME_ALPHABETICAL = new Comparator<Team>() {
        @Override public int compare(Team lhs, Team rhs) {
            return lhs.name.compareTo(rhs.name);
        }
    };
}
```

Notice the `BY_NAME_ALPHABETICAL` `Comparator` - that's the one we'll use to sort the teams as 
required.

Next is the `TeamsAdapter` class, which is pretty straightforward:

```java
public class TeamsAdapter extends RecyclerView.Adapter<TeamsAdapter.ViewHolder> {

    private final LayoutInflater layoutInflater;

    private final List<Team> teams;

    public TeamsAdapter(LayoutInflater layoutInflater) {
        this.layoutInflater = layoutInflater;
        this.teams = new ArrayList<>();
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        return new ViewHolder(layoutInflater.inflate(R.layout.row_team, parent, false));
    }

    @Override
    public void onBindViewHolder(ViewHolder holder, int position) {
        Team team = teams.get(position);
        holder.teamLogo.setImageResource(team.logoRes);
        holder.teamName.setText(team.name);
    }

    @Override public int getItemCount() {
        return teams.size();
    }

    public void setTeams(List<Team> teams) {
        this.teams.clear();
        this.teams.addAll(teams);
        notifyItemRangeInserted(0, teams.size());
    }

    public List<Team> getTeams() {
        return Collections.unmodifiableList(teams);
    }

    static class ViewHolder extends RecyclerView.ViewHolder {

        ImageView teamLogo;
        TextView teamName;

        public ViewHolder(View itemView) {
            super(itemView);
            teamLogo = (ImageView) itemView.findViewById(R.id.team_logo);
            teamName = (TextView) itemView.findViewById(R.id.team_name);
        }
    }
}
```

The `row_team` layout can be found [here][row-team]. Now let's add the code, that will create `Team`
objects and initialize our `TeamAdapter`, to `TeamActivity`:

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    RecyclerView teamsRecyclerView = (RecyclerView) findViewById(android.R.id.list);
    teamsRecyclerView.setLayoutManager(new LinearLayoutManager(this));

    TeamsAdapter teamsAdapter = new TeamsAdapter(LayoutInflater.from(this));
    teamsAdapter.setTeams(createTeams());
    teamsRecyclerView.setAdapter(teamsAdapter);
}

private List<Team> createTeams() {
    List<Team> teams = new ArrayList<>();
    String[] teamNames = getResources().getStringArray(R.array.team_names);
    TypedArray teamLogos = getResources().obtainTypedArray(R.array.team_logos);
    for (int i = 0; i < teamNames.length; i++) {
        Team team = new Team(teamNames[i], teamLogos.getResourceId(i, -1));
        teams.add(team);
    }
    teamLogos.recycle();
    Collections.sort(teams, Team.BY_NAME_ALPHABETICAL);
    return teams;
}
```

We're using the `Team.BY_NAME_ALPHABETICAL` here to properly sort our items before passing them to 
the adapter.

Please fill the gaps in the code by checking the sample on [GitHub][github].

And we're done! Now you can run the test either by right-clicking the `TeamsActivityTest` class and 
selecting the "Run" command, or running the following from the command line:

```
./gradlew connectedAndroidTest
```

The test should run fine, but in case they fail - you'll usually have a pretty helpful output from 
Espresso to help you debug the issue.

Now, we have an acceptance test written in Espresso, that automatically checks that our feature has 
been implemented correctly. As already mentioned, the complete sample code for this project is 
available on [GitHub][github].

[espresso]: https://google.github.io/android-testing-support-library/docs/espresso/index.html
[acceptance-tests]: https://en.wikipedia.org/wiki/Acceptance_testing#Acceptance_testing_in_extreme_programming
[github]: https://github.com/Egorand/android-espresso-sorted-list
[large-test]: https://developer.android.com/reference/android/test/suitebuilder/annotation/LargeTest.html
[activity-instrumentation-test-case-2]: https://developer.android.com/reference/android/test/ActivityInstrumentationTestCase2.html
[activity-test-rule]: https://developer.android.com/reference/android/support/test/rule/ActivityTestRule.html
[matcher]: https://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matcher.html
[guava]: https://github.com/google/guava
[ordering]: https://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Ordering.html
[tdd]: https://en.wikipedia.org/wiki/Test-driven_development
[activity-main]: https://github.com/Egorand/android-espresso-sorted-list/blob/master/app/src/main/res/layout/activity_main.xml
[row-team]: https://github.com/Egorand/android-espresso-sorted-list/blob/master/app/src/main/res/layout/row_team.xml
