---
layout: post
title:  "Testing Reactive Presenters: Destructure Your ViewModel Lists!"
date:   2018-05-10 00:00:00 -0400
tags: [android, kotlin, testing]
---
Back in the early days of Kotlin adoption on Android, many people started writing their unit tests 
using Kotlin. Even though the primary goal of this approach was to have a possibility to enjoy 
Kotlin without letting it slip into the actual production code, it turned out that Kotlin is a great 
language for writing descriptive unit tests, due to its conciseness and elegancy. I still regularly 
discover new ways of applying Kotlin's numerous features to improve the code I write, and I'd like 
to share a neat little testing trick that I've come across just recently.

Imagine the following setup:

```kotlin
data class PageViewModel(
  val title: String? = null,
  val text: String? = null,
  val loading: Boolean = false
)

class PagePresenter(
  private val args: PageScreenArgs,
  private val pageService: PageService,
  private val backgroundScheduler: Scheduler
) {

  fun subscribe(): Observable<PageViewModel> = buildViewModel()

  private fun buildViewModel(): Observable<PageViewModel> {
    return pageService.loadPage(args.pageId)
        .map { page -> PageViewModel(title = page.title, text = page.text) }
        .startWith(PageViewModel(loading = true))
        .subscribeOn(backgroundScheduler)
  }
}
```

First, we've got a ViewModel called `PageViewModel`, which is based on an entity called `Page`. We 
use two flavors of `PageViewModel`: `PageViewModel(loading = true)`, which will tell our View to 
display a loading spinner, and `PageViewModel(title = page.title, text = page.text)`, which carries 
the actual data that the View will render.

Now let's decrypt the RxJava magic that happens in our Presenter: first, we load a `Page` using the 
`PageService` and map it into a `PageViewModel`. The `startsWith` operator allows us to specify an 
element that will appear as the first element in the stream and will be emitted as soon as the View 
subscribes to the Presenter. This behavior matches our use-case: we want to show the spinner as soon 
as the View is displayed, and replace it with the actual data as soon as it's available. Awesome!

Now, let's unit-test this logic! Here's the test case:

```kotlin
@Test fun `builds correct view model`() {
  val presenter = createPresenter(pageId = "page-0")
  `when`(pageService.loadPage(id = "page-0"))
      .thenReturn(Observable.just(Page(title = "Hello world!", text = "Lorem ipsum")))

  val values = presenter.subscribe().test().values()
  assertThat(values).hasSize(2)
  val loading = values[0]
  assertThat(loading.loading).isTrue()
  val page = values[1]
  assertThat(page.title).isEqualTo("Hello world!")
  assertThat(page.text).isEqualTo("Lorem ipsum")
}
```

`test()` is an extremely useful method that returns a `TestObserver`, a utility class that makes 
testing Rx streams easy. The method we're using is `values()`, which simply gives us all events as a 
`List<T>`. We then verify that the list has the correct size and proceed with checking conditions on 
each of the ViewModels we expect to get. There's nothing wrong with this code, but there's a nice 
Kotlin feature that can make it more elegant - list destructuring! Check this out:

```kotlin
@Test fun `builds correct view model`() {
  val presenter = createPresenter(pageId = "page-0")
  `when`(pageService.loadPage(id = "page-0"))
      .thenReturn(Observable.just(Page(title = "Hello world!", text = "Lorem ipsum")))

  val (loading, page) = presenter.subscribe().test().values()
  assertThat(loading.loading).isTrue()
  assertThat(page.title).isEqualTo("Hello world!")
  assertThat(page.text).isEqualTo("Lorem ipsum")
}
```

Here the first and second elements in the list will automatically get assigned to `loading` and 
`page` respectively, so we won't have to query them explicitly. **A word of caution**: list 
destructuring doesn't perform the size check for you. It will still work if you're declaring fewer 
variables then the number of elements in the list:

```kotlin
val (loading) = presenter.subscribe().test().values() // Works!
```

However, if the number of variables is bigger then the number of elements in the list - you'll get 
hit with an `ArrayIndexOutOfBoundsException`:

```kotlin
val (loading, page, third) = presenter.subscribe().test().values() // Mmm nope
```

`TestObserver` to the rescue:

```kotlin
@Test fun `builds correct view model`() {
  val presenter = createPresenter(pageId = "page-0")
  `when`(pageService.loadPage(id = "page-0"))
      .thenReturn(Observable.just(Page(title = "Hello world!", text = "Lorem ipsum")))

  val testObserver = presenter.subscribe().test()
  testObserver.assertValueCount(2)

  val (loading, page) = testObserver.values()
  assertThat(loading.loading).isTrue()
  assertThat(page.title).isEqualTo("Hello world!")
  assertThat(page.text).isEqualTo("Lorem ipsum")
}
```

This version properly checks the preconditions and destructures the list to perform asserts on 
individual elements.

Enjoy!

Thanks to [Aashni][aashni] and [Alec][alec], my colleagues at Cash App, for reviewing this article.

[aashni]: https://twitter.com/aashnisshah
[alec]: https://twitter.com/Strongolopolis
