---
layout: post
title:  "Architecture related decisions in Android - The Rest"
date:   2024-09-04 14:10:00 +0000
categories: android architecture
mermaid: true
---

![starting-image](/assets/img/posts/20240904_architecture_rest.jpg){: width="800" height="535" }

As you can see from the previous articles, architecture-related decisions can have serious implications. I imagine these decisions as a multidimensional sliding puzzle. The one in the above picture is two-dimensional. But our decisions can be much more complex. In this article, I will wrap up the series with a few more decisions we might have to make. These are only some examples I had in mind while writing this article. There are endless examples anyone can come up with.

*Other parts in the series:*<br>
[Architecture related decisions in Android - Introduction]<br>
[Architecture related decisions in Android - Error handling and Monads]<br>
[Architecture related decisions in Android - Mapping]<br>
[Architecture related decisions in Android - Response and Reply classes]<br>
Architecture related decisions in Android - The rest (this article)

## App structure, packaging, and modularisation

If you started Android development as long ago as me, then you remember that we started **packaging by type**. We had packages for activities, interfaces, utils, and so on. This quickly proved to be unmaintainable, and people told you not to package by layer (they meant types), but by feature.

Fine. But then Clean Architecture came along, and we separated the layers. So how should we package? **By feature or layer**? My answer is, as per usual, it depends, this time on many-many things.

But before I go into details, let me clarify that while we **shouldn't hunt for a perfect, one-fits-all solution**, we **shouldn't settle** for anything that comes to mind first. That would be very costly. Instead, we should make the best possible (and quick) decision on something that works best in our situation, based on the constraints and requirements, and then reevaluate again regularly.

Here is a non-exhaustive list of things that will influence your decision: **project size** (now and projected in the future), **modularisation** state, your **preference**, **complexity** of the project, **build times**, **reusability**, **maintainability**, **scalability**, etc.

Most importantly, we need to understand that modern architecture is a **matrix structure**. The first layer of grouping equally could be layers and features. But because the packaging by feature principle has a lot of value to it, that's what we have to try to prioritise. The source of this value is **cohesion**: classes that are used together should be packaged together.

But then we rarely start a new project with multiple features. The best way I can imagine to start a Minimum Viable Product is to start with a single feature with separated concerns, meaning **separated layers**, like domain, data, and presentation, starting with the domain.

```mermaid
block-beta
  columns 1
  a[":data"]
  space
  b[":domain"] 
  space
  c[":presentation"]
  a-->b
  c-->b
  classDef new fill:#f66
  class a new
  class c new
```

This way the project can **grow in two directions independently**: horizontally by adding features and vertically by implementing the data and presentation layers in the feature.

```mermaid
block-beta
  columns 5
  block: movies
    columns 1
    r([":movies"])
    space
    a[":data"]
    space
    b[":domain"] 
    space
    c[":presentation"]
    a-->b
    c-->b
  end
  space
  block: tv
    columns 1
    s([":tv"])
    space
    d[":data"]
    space
    e[":domain"] 
    space
    f[":presentation"]
    d-->e
    f-->e
  end
  space
  block: search
    columns 1
    t([":search"])
    space
    g[":data"]
    space
    h[":domain"] 
    space
    i[":presentation"]
    g-->h
    i-->h
  end
  movies-- "new" -->tv
  tv-- "new" -->search
  classDef new fill:#f66
  class s new
  class t new
```

And this is when the fun part begins.

As the **complexity grows**, you notice that you have to share some common logic. Some repositories and use cases need to be reused at multiple places. Your test mothers needed for both unit tests and UI tests. You add shared modules to your architecture.

```mermaid
block-beta
  columns 5
  a([":movies"])  
  space
  b([":tv"])
  space
  c([":search"])
  space
  d([":shared"]):5
  a-->d
  b-->d
  c-->d
```

You also want to add **more abstractions**, like interface modules for your datasources.

```mermaid
block-beta
  columns 1
  e[":data"]
  space
  d[":data-interface"]
  space
  a[":repository"]
  space
  b[":domain"] 
  space
  c[":presentation"]
  a-- "Impl" -->b
  c-->b
  a-->d
  e-- "Impl" -->d
  classDef planned fill:#6f6
  class d planned
  class e planned
```

But it turns out that adding more modules and dependency injection—at least if it's compile time, like Dagger—increases the **build times** significantly. But when you decide to decrease the number of modules, major releases in Kotlin and Gradle make adding new modules an improvement again.

And then other complexities show up. I discussed these earlier in the series and will discuss them later in this article. Your decisions look more and more random and preferential. As the project grows, but your understanding cannot keep up with it, it is just impossible to find the "optimal,"  the "best" solution. So I recommend not trying to do that, but **strive for improvements**, where it makes sense and it's not too expensive.

In conclusion, you have to **find the balance** between increasing cohesion, maintainability, and scalability (by adding more abstractions and modules) and decreasing complexity and build times (by removing abstractions and merging modules). It is often beneficial to predict some of the future challenges, like rapid growth or reusable features, or to anticipate improvements like the one above for tooling.

## Navigation

The navigation library you choose, the top-level navigation structure you choose, or the style you work with them will have a great influence on other parts of your architecture, like how you pass down or inject your dependencies, or how you structure your modules and packages.

More often than not, any change in the first group will influence what the optimal solution for the second group will be.

For example, when I switched to [Decompose][decompose] for my Companies House project, I had to realise that the dependencies needed to be injected at the top level and passed down. This made the decision over dependency frameworks less important, and Koin or manual DI might be a better solution for this case. Not to mention that the presentation layer needed to be written very differently. The individual parts had a lot of boilerplate, but it made it easier to use that code on multiple platforms through Kotlin Multiplatform.

## Where to use concurrency

I only relatively recently learnt that you **shouldn't do your concurrency in the presentation layer**, in your ViewModels. But now I **moved it even further down** in the hierarchy, from the Repositories to the **DataSources**. Here is why.

Google's [Coroutines Best Practices][google-coroutines-best-practices] document says that "Suspend functions should be main-safe, meaning they're safe to call from the main thread." And then it gives you an example where they make a misterious call from a Repository. No dependencies, and the details are left out.

Elsewhere, in their [architecture guide][architecture-guide] they write that "If a type is performing long-running blocking work, it should be responsible for moving that computation to the right thread."

This is a bit fuzzy, isn't it? What is a Repository then? What is a type exactly? I assume the Repository will have a few DataSources, which are the "type" stores, meaning the abstractions over Retrofit, Room, Ktor, Apollo, Realm, etc.

Does this mean that instead of Repositories, the threading should be in the DataSources? After all, at this point the internet is full of advice that the Repos should have the threading. It turns out that mostly you can get away with—and you probably **should—not using any threading in your repos**.

**Room** does not allow database access on the main thread, and they have a [tutorial][room-async] on writing asynchronous queries.

**Retrofit** internally dispatches on a thread, as does **Realm** for writing, so for these you shouldn't switch threads in the repo. These will run on a **background** thread:

```kotlin
@GET(URL_GENRE)
suspend fun getGenres(
    @Query("api_key") apiKey: String = BuildConfig.TMDB_API_KEY,
    @Query("language") language: String? = "en",
    @Header("If-None-Match") ifNoneMatch: String = ""
): Response<GenreReply>
```
{: file='FlickSlateService.kt'}

```kotlin
override suspend fun insertGenres(genres: List<Genre>) {
    realm.write {
        genres.map { copyToRealm(it.toDbo(), UpdatePolicy.ALL) }
    }
}
```
{: file='GenreDao.kt'}

For cases we return a Flow in Realm, we can use ***flowOn***:

```kotlin
override fun getGenres(): Flow<List<Genre>> {
    return realm
        .query(GenreDbo::class)
        .asFlow()
        .map { dbo -> dbo.list.map { it.toGenre() } }
        .flowOn(ioContext)
}
```
{: file='GenreDao.kt'}

When we return a single item using find(), we need to use withContext() because find is synchronous:

```kotlin
override suspend fun getEtag(): String = withContext(ioContext){
    return@withContext realm.query<EtagDbo>("id = $0", "genres").first().find()?.etag ?: ""
}
```
{: file='GenreDao.kt'}

So to summarise, instead of using concurrency in the Repositories (not to mention ViewModels), I recommend doing it in the DataSources. It makes our lives easier with testing, and generally by moving the hard work where it belongs.

## How to unify ui states

Should we use **separate flow for the items in our state** or a **single sealed class state** object?
This is a subproblem after choosing between MVVM+ (which unifies UI states) and MVI, and neither of the above seems to be a good solution. There are many caveats with both, and the answer is somewhere in between. But I do not even go into details about it, only delegate to the experts. It's worth checking out both links:

[How to safely update state in your Kotlin apps by Nikita Vaizin (Nek.12)][https://proandroiddev.com/how-to-safely-update-state-in-your-kotlin-apps-bf51ccebe2ef]

[Sealed Classes for UI State are an ANTI-PATTERN - Here's why! by Philipp Lackner][https://www.youtube.com/watch?v=NG0PPt-CaYE]

They offer not one, but two different solutions to the above problem: **a sealed interface with "State Families"**, or a **data class with "some" sealed classes inside**. At the time of writing, I haven't decided which one I should prefer, but they seem to solve the problem. As per usual, they could be **different solutions for separate problems**, or they could be **alternative solutions to the same problem**. I'm not sure yet in this case.

## Dependency injection

Dependency injection can influence your architecture in a few different ways.

#### Ways of inversion of control

The most important factor is your decision on how you will provide the dependencies to your classes.

With **manual dependency injection**, you write a lot of boilerplate code (such as factories), which can be error-prone. You also have to manage the scope and lifecycle of the containers yourself, optimising and discarding containers that are no longer needed in order to free up memory. Doing this incorrectly can lead to subtle bugs and memory leaks in your app. This is from Google's [manual DI article][manual-di].

**Service locators** are harder to test and they are hiding dependencies, so nowadays they are considered an antipattern, at least for Android development.

That's why we generally use **Dependency Injection** using **Dagger**, **Koin**, or **kotlin-inject** (or a few other frameworks).

#### Your skills

The features of the libraries do not matter if you do not know about them or you use them the wrong way. For example, if you don't know what Singleton implies in Dagger and you use Singleton annotation without actually applying it. True story :).

#### Testing

Incorrect architecture will result in problems around tests. Dagger and Hilt require a lot of care in architecture because it could lead to slow builds, slow test execution, and other problems down the line.

When I ran into these problems, I was able to come up with a solution, but I'm not sure if it was the best. So for now, I just recommend you read [this article][hilt-testing] and draw your own conclusions.

## Naming

Naming is of utmost importance. It is so important that I will write a separate article about it later.

## Architecture Decision Records

An architecture decision record (ADR) is a document that captures an important architecture decision made along with its context and consequences. You can document all your decisions that you made based on this article series and others.

Go into more details under this link:

[https://github.com/joelparkerhenderson/architecture-decision-record][adr]

## Conclusion

This concludes my article series about architecture decisions. I hope you enjoyed reading it as much as I enjoyed writing it. I have to admit it's much harder to write about them than make the actual decisions. The articles in this series are not as coherent as I'd like them to be.

I learnt a lot during this journey. The most important lesson is that this is a never-ending journey. There is always a lot to learn, so it will be interesting to look back on this series in a few years. Maybe I will laugh about my naiveté and my decisions. But I will definitely praise the courage it took me to actually put these articles out.

[Architecture related decisions in Android - Introduction]: http://localhost:4000/posts/architecture-related-decisions-introduction/
[Architecture related decisions in Android - Error handling and Monads]: http://localhost:4000/posts/architecture-related-decisions-error-handling-and-monads/
[Architecture related decisions in Android - Mapping]: http://localhost:4000/posts/architecture-related-decisions-mapping/
[Architecture related decisions in Android - Response and Reply classes]: http://localhost:4000/posts/architecture-related-decisions-response-classes/
[google-coroutines-best-practices]: https://developer.android.com/kotlin/coroutines/coroutines-best-practices
[architecture-guide]: https://developer.android.com/topic/architecture
[room-async]: https://developer.android.com/training/data-storage/room/async-queries
[https://proandroiddev.com/how-to-safely-update-state-in-your-kotlin-apps-bf51ccebe2ef]: https://proandroiddev.com/how-to-safely-update-state-in-your-kotlin-apps-bf51ccebe2ef
[https://www.youtube.com/watch?v=NG0PPt-CaYE]: https://www.youtube.com/watch?v=NG0PPt-CaYE
[manual-di]: https://developer.android.com/training/dependency-injection/manual
[decompose]: https://arkivanov.github.io/Decompose/
[hilt-testing]: https://medium.com/androiddevelopers/hilt-testing-best-practices-in-the-mad-skills-series-8186a57eee2c
[adr]: https://github.com/joelparkerhenderson/architecture-decision-record