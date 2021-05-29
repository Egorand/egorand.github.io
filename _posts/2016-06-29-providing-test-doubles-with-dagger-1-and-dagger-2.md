---
layout: post
title:  "Providing test doubles with Dagger 1 and Dagger 2"
date:   2016-06-29 00:00:00 -0400
tags: [android, dagger, java, testing]
---
Dagger is a popular dependency injection solution, which works very well with Android, and at the 
moment is probably a must for any more or less complicated Android project. However, there are 
things about Dagger that you need to wrap your head around when you're beginning to use it. Also, 
there are things that don't look that straightforward even after you've used Dagger for some time. 
One of these things is the way to configure the modules to provide fakes in test mode, and there's 
been a lot of confusion around this topic. Another reason for the confusion is the way 
[Dagger 1][dagger-1] and [Dagger 2][dagger-2] handle overriding - those are a bit different. It 
hasn't been a long time since I've refactored Dagger 2 modules in my current project, so I decided 
to dive a little deeper into the problem and describe the solution that I believe is the correct 
one. Welcome on board!

### Overriding modules in Dagger 1

Let's first see how the things worked back in the days of Dagger 1. Imagine we have the following 
Dagger 1 module called `AppModule`:

```java
@Module(library = true, injects = GreetingGenerator.class)
public class AppModule {

    private final DaggerOverridesApp app;

    public AppModule(DaggerOverridesApp app) {
        this.app = app;
    }

    @Provides @AppScope Context provideAppContext() {
        return app;
    }

    @Provides GreetingGenerator provideGreetingGenerator() {
        return new HelloGreetingGenerator();
    }
}
```

The module provides the `@AppScope Context`, and an implementation for an interface called 
`GreetingGenerator`. In Dagger 1 we would need to create an `ObjectGraph` using this module, and 
that's what we'll do in the application class:

```java
public class DaggerOverridesApp extends Application {

    private ObjectGraph graph;

    @Override public void onCreate() {
        super.onCreate();

        graph = ObjectGraph.create(new AppModule(this));
    }

    public ObjectGraph graph() {
        return graph;
    }
}
```

Now, whenever we need a `GreetingGenerator`, we can get one like this:

```java
ObjectGraph graph = ((DaggerOverridesApp) getApplication()).graph();
greetingGenerator = graph.get(GreetingGenerator.class);
```

Now say we decided to write some Espresso tests and want to substitute the implementation of 
`GreetingGenerator` with a mock. Let's see how we can achieve it with Dagger 1.

### An OK-ish approach

In Dagger 1 we can create a module with the `overrides` parameter:

```java
@Module(injects = GreetingGenerator.class, overrides = true)
public class MockGreetingModule {

    private final GreetingGenerator mockGreetingGenerator;

    public MockGreetingModule() {
        this.mockGreetingGenerator = mock(GreetingGenerator.class);
    }

    @Provides GreetingGenerator provideGreetingGenerator() {
        return mockGreetingGenerator;
    }
}
```

Compiler will override any `@Provides` methods defined in `AppModule` with the ones we supply in 
`MockGreetingModule`. Next we'll create the test version of the application class and will create 
the graph as follows:

```java
public class TestDaggerOverridesApp extends DaggerOverridesApp {

    private ObjectGraph graph;

    @Override public void onCreate() {
        super.onCreate();

        graph = ObjectGraph.create(
                new AppModule(this),
                new MockGreetingModule());
    }

    @Override public ObjectGraph graph() {
        return graph;
    }
}
```

All test classes that query a `GreetingGenerator` from this application flavor will now get a mock. 
This is a working solution, but apparently not a very clean one. Let's see what the Javadoc for 
`overrides` says:

> This is a dangerous feature as it permits binding conflicts to go unnoticed. It should only be 
> used in test and development modules.

Well, we're in test mode, so we're kind of fine, but we can do better!

### A better approach

Thing is, the `ObjectGraph` doesn't care how many modules you pass in as long as all `@Inject` 
targets are satisfied. That means that the logic of overriding can (and should) be implemented with 
**pluggable modules**. First of all, let's separate out the logic for providing the 
`GreetingGenerator` into its own module:

```java
@Module(injects = GreetingGenerator.class)
public class GreetingModule {

    @Provides GreetingGenerator provideGreetingGenerator() {
        return new HelloGreetingGenerator();
    }
}
```

This module will now be solely responsible for satisfying the `GreetingGenerator` dependency, hence 
it will be easy to replace. Now we'll be creating the `ObjectGraph` in the application class as 
follows:

```java
graph = ObjectGraph.create(new AppModule(this), new GreetingModule());
```

Now we can remove the `overrides` from the `MockGreetingModule`:

```java
@Module(injects = GreetingGenerator.class)
public class MockGreetingModule {

    private final GreetingGenerator mockGreetingGenerator;

    public MockGreetingModule() {
        this.mockGreetingGenerator = mock(GreetingGenerator.class);
    }

    @Provides GreetingGenerator provideGreetingGenerator() {
        return mockGreetingGenerator;
    }
}
```

and the logic for creating the `ObjectGraph` inside the test application class will stay the same:

```java
graph = ObjectGraph.create(new AppModule(this), new MockGreetingModule());
```

Voilà! You see that we **plugged** `MockGreetingModule` to provide `GreetingGenerator`, and the 
Dagger compiler is happy. Let's memorize this concept and see how we can apply it in Dagger 2.

### Overriding modules in Dagger 2

In Dagger 2 the graph is created by defining a set of interfaces marked with the `@Component` 
annotation. An `@Component` interface is essentially a contract that declares a set of dependencies 
it can provide. The component relies on a set of modules to implement the logic of providing 
dependencies, and the modules must be declared explicitly. Let's introduce a component into our 
example:

```java
@AppScope
@Component(modules = {AppModule.class})
public interface AppComponent {

    @AppScope Context appContext();

    GreetingGenerator greetingGenerator();
}
```

The application class implementation will change to the following:

```java
public class DaggerOverridesApp extends Application {

    private AppComponent appComponent;

    @Override public void onCreate() {
        super.onCreate();

        appComponent = DaggerAppComponent.builder()
                .appModule(new AppModule(this))
                .build();
    }

    public AppComponent appComponent() {
        return appComponent;
    }
}
```

and we'll be querying the `GreetingGenerator` like this:

```java
AppComponent component = ((DaggerOverridesApp) getApplication()).appComponent();
greetingGenerator = component.greetingGenerator();
```

Alright, now let's try to create a test module that will allow us to supply a mocked version of 
`GreetingGenerator`. First, a hacky approach.

### A hacky approach

As you can see from the code snippet above, `DaggerAppComponent` builder expects us to pass in a 
module of type `AppModule`, so our test module will have to have this type. However, if we'll just 
go on and extend `AppModule`:

```java
@Module
public class MockAppModule extends AppModule {

    final GreetingGenerator mockGreetingGenerator = mock(GreetingGenerator.class);

    public MockAppModule(DaggerOverridesApp app) {
        super(app);
    }

    @Provides public GreetingGenerator provideGreetingGenerator() {
        return mockGreetingGenerator;
    }
}
```

the Dagger compiler will complain:

```java
Error:(38, 40) error: @Provides methods may not override another method.
Overrides: @Provides me.egorand.daggeroverrides.model.GreetingGenerator
me.egorand.daggeroverrides.di.module.AppModule.provideGreetingGenerator()
```

Additionally, Dagger 2 doesn't support `overrides` anymore, so there's no magic parameter that will
let us satisfy the compiler. However, there's a trick to fool the compiler by creating an
anonymous module class at runtime:

```java
public class TestDaggerOverridesApp extends DaggerOverridesApp {

    private AppComponent appComponent;

    @Override public void onCreate() {
        super.onCreate();

        appComponent = DaggerAppComponent.builder()
                .appModule(new AppModule(this) {

                    final GreetingGenerator mockGreetingGenerator = mock(GreetingGenerator.class);

                    @Override public GreetingGenerator provideGreetingGenerator() {
                        return mockGreetingGenerator;
                    }
                })
                .build();
    }

    @Override public AppComponent appComponent() {
        return appComponent;
    }
}
```

And this will do the job! Again however - we can do better!

### A better approach

First of all, let's define a separate component that will be responsible for providing the 
`GreetingGenerator` only:

```java
@Component(modules = GreetingModule.class)
public interface GreetingComponent {

    GreetingGenerator greetingGenerator();
}
```

Now, as I mentioned, a component in Dagger 2 is a contract that relies on a set of modules to 
provide a set of dependencies. We can go on and extend the component interface, which semantically 
would mean that we're declaring a different contract that satisfies the same set of  dependencies, 
but is able to rely on a different set of modules - which is exactly what we need:

```java
@Component(modules = MockGreetingModule.class)
public interface MockGreetingComponent extends GreetingComponent {
}
```

`MockGreetingComponent` can provide an implementation of `GreetingGenerator` and it relies on 
`MockGreetingModule` to create it. We can instantiate it in the test version of the application 
class as follows:

```java
mockGreetingComponent = DaggerMockGreetingComponent.builder()
                .mockGreetingModule(new MockGreetingModule())
                .build();
```

Given that `MockGreetingComponent` is a subtype of `GreetingComponent`, this substitution will be 
transparent for the calling code. Way to go!

If you'd like to understand the concept better and experiment with the setup, the 
[GitHub repo][github] contains the following 4 branches:

- [`dagger1-overrides`][github-dagger1-overrides] - demonstrates the creation of a test module using 
  the `overrides` parameter in Dagger 1
- [`dagger1`][github-dagger1] - demonstrates a clean solution using separate modules in Dagger 1
- [`dagger2-overrides`][github-dagger2-overrides] - demonstrates a hacky solution to override a 
  module in Dagger 2
- [`dagger2`][github-dagger2] - demonstrates a clean solution using separate modules in Dagger 2

### Conclusion

As we've seen, Dagger can be tricky, and there are some non-trivial semantics involved that can take 
time to grasp. However, a clean Dagger configuration can do wonders for the testability and 
maintainability of your Android project, so it's worth spending some time to experiment with 
Dagger's features. Hopefully this article will bring you a tiny step closer to mastering this 
powerful tool.

Cheers!

[dagger-1]: https://square.github.io/dagger/
[dagger-2]: https://google.github.io/dagger/
[github]: https://github.com/Egorand/android-dagger-overrides
[github-dagger1-overrides]: https://github.com/Egorand/android-dagger-overrides/tree/dagger1-overrides
[github-dagger1]: https://github.com/Egorand/android-dagger-overrides/tree/dagger1
[github-dagger2-overrides]: https://github.com/Egorand/android-dagger-overrides/tree/dagger2-overrides
[github-dagger2]: https://github.com/Egorand/android-dagger-overrides/tree/dagger2
