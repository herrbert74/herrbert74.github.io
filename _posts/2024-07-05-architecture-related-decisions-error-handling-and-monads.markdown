---
layout: post
title:  "Architecture related decisions in Android - Error handling and Monads"
date:   2024-07-05 17:10:00 +0000
categories: android architecture
---

![starting-image](/assets/img/posts/20240604_error_handling.jpg){: width="512" height="512" }

<a href="https://androidweekly.net/issues/issue-630"><img alt="Featured in Android Weekly Issue 630" src="/assets/img/posts/20240705_badge.svg" width="252" height="20"></a>

Other parts in the series:<br>
[Architecture related decisions in Android - Introduction]<br>
Architecture related decisions in Android - Error handling and Monads (this article)<br>
[Architecture related decisions in Android - Mapping]<br>
*Architecture related decisions in Android - Response and Reply classes* (coming soon)<br>
*Architecture related decisions in Android - The rest* (coming soon)


I wrote this article as two separate articles, but it proved impossible to talk about error handling or monads without the other. For me, the two concepts are intertwined. So I will introduce both, tell you how they affect your decisions in the app stack, and finally, how I am thinking about them at the moment.

I learned a lot writing this article; it changed how I use these concepts. There were a lot of twists and turns, so don’t take anything for granted and make a decision for yourself.

## Error handling

An important background for error handling in Kotlin is [Roman Elizarov's post about the topic][elizarov-kotlin-exceptions].

Roman clearly indicates that Exceptions in Kotlin should be used to handle logic errors, meaning developer errors, and as such, unexpected Exceptions should not be caught.

However, he advocates using a sealed class hierarchy to handle expected API exceptions. It's not very clear to me what he means by handling input-output errors at the top level and if it applies to Android.

According to [this][typed-error-handling] article, which compares various error handling techniques, sealed classes can go quickly out of hand, and I tend to agree with this. Using context receivers with the Arrow library looks promising, but I haven't looked into that as I went down the monad route for now.

## What are monads?

![monad-image](/assets/img/posts/20240602_monads.jpg){: width="512" height="512" }

*(The image was for the other article, but I liked it too much not to use it.)*

Monad is a generic concept that helps deal with side effects when doing operations between pure functions. I recommend reading Adam Bennett's article about this concept. This article introduced it to me as well:

[https://adambennett.dev/2020/05/the-result-monad/][what-are-monads]

If you want to learn more about it in general, maybe you could read its Wikipedia entry:

[https://en.wikipedia.org/wiki/Monad_(functional_programming)][monads-wikipedia]

For reasoning about naming monads and other classes, please refer to my earlier article:

[https://herrbert74.github.io/posts/caching-strategies-in-android/#monads-and-other-terminology][monad-naming]

## Should we use monads? The dangers of runCatching

> Do not use runCatching in itself to catch input-output errors.
{: .prompt-danger }

Many, like [this article][runCatching-is-problematic] advocate against **runCatching** because it's a blanket solution that catches everything.

For this reason, I created a new overloaded version of runCatching that will only catch certain errors and rethrow some others. I updated my previous example project from my strategies article. For more details, check [KotlinResult.kt][flickslate-kotlin-result] and [ToFailureMappers.kt][flickslate-tofailuremappers]. This is how it looks:

```kotlin

inline fun <V> runCatchingApi(block: () -> V) = runCatching(block)
	.mapError { it.handle() }

inline fun <V> runCatchingUnit(block: () -> V) = runCatching(block)
	.onFailure { Timber.d("zsoltbertalan* runCatchingUnit: ${it.message}") }

```
{: file='KotlinResult.kt'}

The **runCatchingApi** function is meant to be used for calls returning a Result, while **runCatchingUnit** will be used for functions returning Unit, where an Exception is expected but needs to be caught without letting the user know. An example could be saving cached data to the database, where the cache is **not critical**.

The above code uses error handling like this. As you can see, there is a **retrofit2.Response** version as well:

```kotlin

fun Throwable.handle() = when (this) {
	is UnknownHostException -> Failure.UnknownHostFailure
	is HttpException -> Failure.ServerError
	is IOException -> Failure.IoFailure
	else -> throw this
}

fun <T> Response<T>.handleCode() = when {
	this.code() == HTTP_BAD_REQUEST -> Failure.ServerError
	this.code() == HTTP_NOT_FOUND -> Failure.ServerError
	this.code() == HTTP_NOT_MODIFIED -> Failure.NotModified
	else -> Failure.UnknownApiError
}

```
{: file='ToFailureMappers.kt'}

It handles and converts errors to Failures, domain-specific errors expressed as a sealed class. These are not strictly domain-specific. Some common exceptions are converted to Failures in the above examples. This way, common and truly domain-specific Exceptions can be handled together.

For the runCatchingApi, I return a shortened typealias, like before, but instead of using a throwable, I'm using Failure, and I named it Outcome for now:

```kotlin

typealias Outcome<T> = Result<T, Failure>

```
{: file='KotlinResult.kt'}

## Which monad library to use?

There are many options to consider, for example:

* [**Kotlin Result**][kotlin-result-standard] from the Kotlin standard library
* [**Kotlin-Result**][kotlin-result]: A confusingly named third-party library 
* [**Arrow**][arrow]
* **Custom**

The [original KEEP][kotlin-result-standard-keep] for the standard Result class states that Kotlin Result is not to be used for domain-specific errors. The first main reason is that it only accepts a **Throwable** parameter for failure. But I want to use my own classes for expected exceptions, which do not need to be throwables. The second is that, among others, they do not provide **flatmap/andThen** extensions, which are essential for more complex cases, which we constantly run into.

This ticket gives you some context on this:<br>
[https://github.com/michaelbull/kotlin-result/issues/59][kotlin-result-differences]

I had a quick look at **Arrow** and **custom** implementations, but I found that **Kotlin-Result** is the easiest to use.

All in all, my recommendation is to use the **Kotlin-Result** library. It offers a lot of transforming and chaining functions, coroutine, and multiplatform support, while now with 2.0, it maintains zero overhead on the happy path.

## Where to create the monad?

I recommend wrapping networking and database operations on the **Repository layer** through the runCatching function. This way, the data layer can deal with the data itself. The Repository layer can merge the various data implementations, for example, by introducing caching strategies. This is the perfect place to introduce monads as well.

Here is an example of using runCatching in my FlickSlate application:

```kotlin

override suspend fun getMovieDetails(movieId: Int): Outcome<MovieDetail> {
    return flickSlateService.runCatchingApi {
        getMovieDetails(movieId = movieId).toMovieDetail()
    }
}

```
{: file='MoviesAccessor.kt'}

You can check out the code [here][runcatching].

An alternative to this is to use a **Retrofit call adapter** and return a monad or a sealed class on the **network layer**. [This library][retrofit-call-adapter-returning-result] uses this approach. It has a lot of merit, as the responsibility may be clearer, but I feel doing it in the repository is more convenient because the transformations look more fluid.

The next step is aggregating, transforming, and chaining the various repository calls. This can be done in use cases or, if you don't use them, in the repository itself. This step might be done in the Presentation layer as well, in ViewModels or Executors in the case of MVI.

## Where to use the monad?

I recommend unwrapping them in the ViewModel/Executor and obviously using it in the UI. The alternative would be to use it in the UI directly, but I see no reason to make such decisions in the UI.

## Summary

This article discusses error handling and monads in Kotlin, focusing on their intertwined nature in the context of Android architecture. It represents my current views on how this should be implemented. I recommend creating Kotlin-Result monads in the Repository layer and using them in your Presentation layer, but not in the UI.

In the next article, I will discuss mapping in the context of Android architecture.

[Architecture related decisions in Android - Introduction]: https://herrbert74.github.io/posts/architecture-related-decisions-introduction/
[Architecture related decisions in Android - Mapping]: https://herrbert74.github.io/posts/architecture-related-decisions-mapping/
[Architecture related decisions in Android - Response and Reply classes]: http://localhost:4000/posts/architecture-related-decisions-response-classes/
[Architecture related decisions in Android - The rest]: http://localhost:4000/posts/architecture-related-decisions-rest/
[elizarov-kotlin-exceptions]: https://elizarov.medium.com/kotlin-and-exceptions-8062f589d07
[kotlin-result-standard-keep]: https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md
[typed-error-handling]: https://betterprogramming.pub/typed-error-handling-in-kotlin-11ff25882880
[retrofit-call-adapter-returning-result]: https://haroldadmin.github.io/NetworkResponseAdapter/
[runCatching-is-problematic]: https://medium.com/sampingan-tech/kotlin-getting-to-knows-with-exceptions-564f8b2bc3c
[what-are-monads]: https://adambennett.dev/2020/05/the-result-monad/
[monads-wikipedia]: https://en.wikipedia.org/wiki/Monad_(functional_programming)
[monad-naming]: https://herrbert74.github.io/posts/caching-strategies-in-android/#monads-and-other-terminology
[kotlin-result]: https://github.com/michaelbull/kotlin-result
[kotlin-result-differences]: https://github.com/michaelbull/kotlin-result/issues/59
[kotlin-result-standard]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/
[arrow]: https://arrow-kt.io/
[runcatching]: https://github.com/herrbert74/FlickSlate/blob/main/app/src/main/java/com/zsoltbertalan/flickslate/data/repository/MoviesAccessor.kt
[flickslate-kotlin-result]: https://github.com/herrbert74/FlickSlate/blob/main/app/src/main/java/com/zsoltbertalan/flickslate/ext/KotlinResult.kt
[flickslate-tofailuremappers]: https://github.com/herrbert74/FlickSlate/blob/main/app/src/main/java/com/zsoltbertalan/flickslate/util/getresult/ToFailureMappers.kt