---
layout: post
title:  "Android Architecture - Fetchers update"
date:   2025-03-23 16:10:00 +0000
categories: android architecture
mermaid: true
---

![starting-image](/assets/img/posts/20241228_fetchers_update.jpg){: width="800" height="535" }

I introduced the fetcher collection calls in [my article about Caching Strategies in Android][caching-getresult] a while ago. You might know these functions as [networkBoundResource][networkBoundResource] or from libraries as Flower.

As I wrote in the article above, I like to use this concept in an opinionated way, **so instead of a library I added these to my [hobby project][FlickSlate] for now**. I will keep it this way until I feel the urge or the need to use this in other projects. Feel free to copy them into your project or a library.

This is a quick update to these functions, as I found a more generic way to do it and added a few minor improvements.

### Moving the mapping to DataSources

As I wrote in my [article about the mapping][Architecture related decisions in Android - Mapping], I couldn't find a consistent solution to this. Well, this is it, if you are OK with the slightly increased complexity.

The most important change is that the fetcher functions do not do the mapping anymore. This was moved down the chain to the newly introduced RemoteDataSources and safeCall functions. The safeCall functions will also deal with the data extraction from HTTP headers.

This is the safeCall, which maps and recovers from exceptions by retrieving the error from the Response:


```kotlin
suspend inline fun <REMOTE, DOMAIN> safeCall(
  crossinline makeNetworkRequest: suspend () -> Response<REMOTE>,
  crossinline mapper: REMOTE.() -> DOMAIN,
): Outcome<DOMAIN> {
  return runCatchingApi {
    makeNetworkRequest()
  }.andThen { response ->
    if (response.isSuccessful) {
      Ok(response.body()!!)
    } else {
      Err(response.handle())
    }
  }.map {
    it.mapper()
  }
}
```
{: file='SafeCall.kt'}

For the FlickSlate project I needed the above, but you might have an API that doesn't have the errors in the error body. In this case you simply could use this:

```kotlin

suspend inline fun <REMOTE, DOMAIN> safeCall(
  crossinline makeNetworkRequest: suspend () -> REMOTE,
  crossinline mapper: REMOTE.() -> DOMAIN,
): Outcome<DOMAIN> {
  return runCatchingApi { makeNetworkRequest() }
    .map { it.mapper() }
}
```
{: file='SafeCallAlternative.kt'}

Finally, this is the version to retrieve metadata from the Response headers:

```kotlin
suspend inline fun <REMOTE, DOMAIN> safeCallWithMetadata(
  crossinline makeNetworkRequest: suspend () -> Response<REMOTE>,
  crossinline mapper: Response<REMOTE>.() -> DOMAIN,
): Outcome<DOMAIN> {
  return runCatchingApi {
    makeNetworkRequest()
  }.andThen { response ->
    if (response.isSuccessful) {
      Ok(response)
    } else {
      Err(response.handle())
    }
  }.map {
    it.mapper()
  }
}
```
{: file='SafeCall.kt'}

And this is an example of how to use the above in a RemoteDataSource.

```kotlin


class PopularMoviesRemoteDataSource @Inject constructor(
  private val moviesService: MoviesService
) : PopularMoviesDataSource.Remote {

  override suspend fun getPopularMovies(etag: String?, page: Int?): Outcome<PagingReply<Movie>> {
    return safeCallWithMetadata(
      { moviesService.getPopularMovies(ifNoneMatch = etag, page = page) },
      Response<MoviesReplyDto>::toMoviesReply
    )
  }

}
```
{: file='PopularMoviesRemoteDataSource.kt'}

Also important to see how we handle the Errors and Exceptions.

```kotlin

/**
 * Handle only expected Exceptions, throw Errors and other Exceptions, like CancellationException
 */
fun Throwable.handle() = when (this) {
  is UnknownHostException -> Failure.UnknownHostFailure
  is HttpException -> {
    val errorJson = this.response()?.errorBody()?.string() ?: ""
    val errorBody = Json.decodeFromString<ErrorBody>(errorJson) Failure.ServerError(errorBody.status_message)
  }
  is IOException -> Failure.IoFailure
  else -> throw this
}

/**
 * Handle all unsuccessful Responses.
 */
fun <T> Response<T>.handle() = when {
  this.code() == HTTP_NOT_MODIFIED -> Failure.NotModified
  else -> {
    val errorJson = this.errorBody()?.string() ?: ""
    Failure.ServerError(errorJson)
  }
}
```
{: file='ToFailureMappers.kt'}

Note that everything is converted to domain-specific Failure, which is a sealed class, and that CancellationExceptions are rethrown, so the view lifecycle is not disturbed.

This way the mapping and error handling happen purely in the data layer, while the repetitive stuff is extracted to generic functions. Pretty neat, eh?

### Repository layer

With the above improvements the fetch calls and the Repositories can be greatly simplified. The Repositories are not responsible for mapping and extracting data from error bodies or metadata.

There isn't any need for separate Response and no-Response versions either.

With this, the **fetchNetworkOnly** strategies could be completely eliminated. If needed, you can call the DataSource directly, and the safeCall inside it will handle everything you need:

```kotlin
class SearchAccessor @Inject constructor(
  private val searchMoviesRemoteDataSource: SearchMoviesRemoteDataSource
) : SearchRepository {

  override suspend fun getSearchResult(query: String, page: Int): Outcome<PagingReply<Movie>> =
    searchMoviesRemoteDataSource.searchMovies(query = query, page = page)

}

```
{: file='SearchAccessor.kt'}

And then the DataSource:

```kotlin
class SearchMoviesRemoteDataSource @Inject constructor(
  private val searchService: SearchService
) : SearchMoviesDataSource.Remote {

  override suspend fun searchMovies(query: String, page: Int?): Outcome<PagingReply<Movie>> {
    return safeCall(
      {
        searchService.searchMovies(
          query = query,
          language = "en",
          page = page
        )
      },
      MoviesReplyDto::toMoviesReply
    )
  }

}
```
{: file='SearchMoviesRemoteDataSource.kt'}

This is how **fetchRemoteFirst** looks now, after removing mapping:

```kotlin
inline fun <DOMAIN> fetchRemoteFirst(
  crossinline fetchFromLocal: () -> Flow<DOMAIN?>,
  crossinline makeNetworkRequest: suspend () -> Outcome<DOMAIN>,
  noinline saveResponseData: suspend (DOMAIN) -> Unit = { },
  retryPolicy: RetryPolicy<Failure> = defaultNoRetryPolicy,
) = flow<Outcome<DOMAIN>> {

  var localData: DOMAIN? = null

  val result = retry(retryPolicy) { makeNetworkRequest() }.andThen { domain ->
    runCatchingUnit { saveResponseData(domain) }
    Ok(domain)
  }.recoverIf(
    { failure ->
      localData = fetchFromLocal().first()
      failure == Failure.NotModified || localData != null
    },
    { localData }
  )

  if (result.isErr || (result.isOk && result.component1() != null)) {
    emitAll(
      flowOf(
        result.map { it!! }
      )
    )
  }
}
```
{: file='fetchRemoteFirst.kt'}

I don't copy **fetchCacheThenRemote** here, you can find it in the project.

As the DataSources return domain classes, we don't have to deal with any Responses and Dtos in the Repositories. This means that the responsibilities of the DataSources and Repositories are very clear: mapping and error handling happen in DataSources. Data preparation and handling always happens in the Repositories. With this, the Repositories are much closer to the domain layer, though still part of the data layer.

I also renamed fetchNetwork* to fetchRemote* to align the naming with DataSources and generic parameters in the fetchers.

### Retry

Another feature I added was handling retries. For this I used another library related to kotlin-result from the same author: [kotlin-retry][kotlin-retry].

As a sidenote, the kotlin-result and kotlin-retry-result libraries switched to inline functions, but these don't work really well in all situations, for example, when using them together with mockK. So for now I use a no-inline version of them: 1.x in the case of kotlin-result and a modified copy of kotlin-retry-result.

There are four retry policies, which can be combined through the plus operator. They are: Backoff, Predicate, Delay and Stop. I tried to add at least one each to the below examples.

```kotlin

val defaultNoRetryPolicy = continueIf<Failure> { false }
val defaultRetryPolicy = constantDelay<Failure>(RETRY_DELAY) + stopAtAttempts(RETRY_ATTEMPTS)
val backoffRetryPolicy = binaryExponentialBackoff<Failure>(min = 10L, max = 5000L) + stopAtAttempts(RETRY_ATTEMPTS)
```

You can add any combinations like above to your project. I used the defaultNoRetryPolicy as default, the backoffRetryPolicy in some of the app usages and the defaultRetryPolicy in the data layer tests.

# Conclusion

With the above improvements the fetcher functions are close to prime time, but I still don't have enough code using them. If you find them useful, or especially if you find bugs in them, please let me know on the project or here in the comments.

[caching-getresult]: https://herrbert74.github.io/posts/caching-strategies-in-android/#getresults-a-collection-of-repository-functions
[networkBoundResource]: https://blog.devgenius.io/android-networking-and-database-caching-in-2020-mvvm-retrofit-room-flow-35b4f897d46a
[FlickSlate]: https://github.com/herrbert74/FlickSlate
[Architecture related decisions in Android - Mapping]: https://herrbert74.github.io/posts/architecture-related-decisions-mapping/
[kotlin-retry]: https://github.com/michaelbull/kotlin-retry