---
layout: post
title:  "Caching Strategies in Android"
date:   2024-05-22 09:10:00 +0000
categories: android caching
mermaid: true
---

![starting-image](/assets/img/posts/20240521.jpg){: width="512" height="512" }

<a href="https://androidweekly.net/issues/issue-624"><img alt="Featured in Android Weekly Issue 624" src="/assets/img/posts/20240521_badge.svg" width="252" height="20"></a>

*Update 1: Added stale-if-error and update-while-navigate and corrected stale-while-revalidate*

*Update 2: Added warning about runCatching*

*Update 3: Added Cache First - Network Parallel strategy*

Recently, I came across a few challenges that involved caching. In everything I do, I strive to understand as much of it as possible. What type of caching is available? How should I decide on strategies? I found information on the technical part, but I couldn't find Android-related instructions. I also ran across a few interesting tidbits that I would like to share.

## Caching types

There are many different types of caching, but in Android, we are mostly interested in ***API caching*** and ***in-memory caching***. There is some overlap between them, but for this article, I will only investigate API caching, meaning the relationship between network calls and database calls, as opposed to in-memory caching to speed up reading or writing data in between API calls and database requests, or in relation to only one of them.

## Caching strategies

First, I came across [this][caching-strategies] article. It concentrates on memory caches in the context of web development, but it still provides a good starting point for my article.

Second, another similar [article][mastering-caching-strategies], with more strategies and a more digestible explanation of how it works, why we need it, and what types of caching are there.

It is worth noting that strategies like Write-Through, Read-Through, Write-Back, and Write-Around Cache do not seem to fit the single responsibility principle and the repository pattern because they make decisions about the cache outside of the repository, but they might be worth exploring later and treated as an extension to the repository pattern if they provide performance benefits. For now, I concentrate on the simpler API caching only strategies.

Third, an Android-specific article about a cache library. My takeaway from here is some of the strategies, but I adopted a different style:

[https://medium.com/@andrei_r/easy-caching-android-kotlin-flow-b824a29e8a77][universal-caching]

From the above I identified the following strategies that are relevant for Android API caching:

### Cache Only

The most basic strategy, which I just mention for completeness. Use this when you can build your cache from a compact source. I used this for a local only quiz game, where I stored the initial data in Protobuf format.

```mermaid
sequenceDiagram
    Repo->>Cache: Request
    Cache-->>Repo: Success
    Cache--XRepo: Failure
```

### Network Only

The other basic strategy, more often used for MVPs, when you quickly want to build something.

```mermaid
sequenceDiagram
    Repo->>Network: Request
    Network-->>Repo: Success
    Network--XRepo: Failure
```

### Network First (aka stale-if-error)

We make a network request first, and only in case the network fails we go to the cache. Similar to [HTTP RFC 5861 stale-if-error][rfc5861].

```mermaid
sequenceDiagram
    participant UI
    Repo->>Network: Request
    alt
    Network-->>Repo: Success
    Repo-->>UI: Emit
    Repo->>Cache: Save
    else
    Network--XRepo: Failure
    Repo->>Cache: Request
    alt
    Cache-->>Repo: Success
    Repo-->>UI: Emit
    else
    Cache--XRepo: Failure
    Repo-->>UI: Emit error
    end
    end
```

### Cache First - Network Second (aka stale-while-invalidate)

For the Cache First strategies I try to highlight the differences between them.

For Cache First - Network Second, we go to the cache first, emit it, then we ***always*** do a network request, and if there is new data, we save and ***emit*** it.

Similar to [HTTP RFC 5861 stale-while-invalidate][rfc5861].

Most commonly used variant, in my opinion.

```mermaid
sequenceDiagram
    participant UI
    Repo->>Cache: Request
    alt
        Cache-->>Repo: Success
        Repo-->>UI: Emit
    else
        Cache-->>Repo: Failure
    end
    rect rgb(250, 223, 220)
        Repo->>Network: Request always
    end
    alt
        Network-->>Repo: Success
        rect rgb(250, 223, 220)
            Repo-->>UI: Emit
        end
        Repo->>Cache: Save
    else
        Network--XRepo: Failure
        rect rgb(250, 223, 220)
            opt cache failed too
                Repo-->>UI: Emit error
            end
        end
    end
```

### Cache First - Network Parallel

We **always** make a network request, but at the same time we **start listening** to cache changes with an **initial emission**. We **only save** the network result into the cache, not delivering it directly to the UI.

It is very similar to Cache First - Network Second, but here the cache is treated as the single source of truth. Technically, this is not even Cache First, but effectively, the cache result—if there is any—is always delivered first.

I use this approach only when multiple requests to the network are expected or when there are multiple contributors to the cache.

```mermaid
sequenceDiagram
    participant UI
    
    par Listen to Flow
        rect rgb(250, 223, 220)
            Repo->>Cache : Start listening
        end
        activate Cache
        alt Cache Success
            rect rgb(250, 223, 220)
                Cache-->>Repo: Initial Emission
            end
            Repo-->>UI: Emit
        else Cache Failure
            Cache--XRepo: Failure
        end
    and Request
        rect rgb(250, 223, 220)
            Repo->>Network : Request always
        end
        activate Network
        alt Network Success
            Network-->>Repo: Success
            deactivate Network
            rect rgb(250, 223, 220)
                Repo->>Cache: Save
                Cache-->>Repo: Emission on separate function
            end
            deactivate Cache
            Repo-->>UI: Emit
        else Network Failure
            Network--XRepo: Failure
            rect rgb(250, 223, 220)
                opt cache failed too
                    Repo-->>UI: Emit error
                end
            end
        end
    end
```

### Cache First - Network for Later (aka update-while-navigate)

Like before, we emit cache and ***always*** do network request, but we ***do not emit*** result, only save it.

Use for less frequently changing data, where live data is less important. It's a mixture of CO and CFNS because it only uses the cache, but every usage triggers a network request and cache refresh.

```mermaid
sequenceDiagram
    participant UI
    Repo->>Cache: Request
    alt
    Cache-->>Repo: Success
    Repo-->>UI: Emit
    else
    Cache-->>Repo: Failure
    end
    rect rgb(250, 223, 220)
    Repo->>Network: Request always
    end
    alt
    Network-->>Repo: Success
    Repo->>Cache: Save
    else
    Network--XRepo: Failure
    rect rgb(250, 223, 220)
    opt cache failed too
    Repo-->>UI: Emit error
    end
    end
    end
```

### Cache First - Network Once

Like before, we emit cache, but ***only*** do network request, if the cache has failed, then we ***always*** emit any result.

It's also a mixture of CO and CFNS with slightly different rules.

Use it when data never changes but only available through an API.

```mermaid
sequenceDiagram
    participant UI
    Repo->>Cache: Request
    alt
    Cache-->>Repo: Success
    Repo-->>UI: Emit
    else
    Cache--XRepo: Failure
    rect rgb(250, 223, 220)
    Repo->>Network: Request on failure
    end
    alt
    Network-->>Repo: Success
    Repo->>Cache: Save
    rect rgb(250, 223, 220)
    Repo-->>UI: Emit
    end
    else
    Network--XRepo: Failure
    rect rgb(250, 223, 220)
    Repo-->>UI: Emit error
    end
    end
    end
```

## GetResults: A collection of Repository functions

After identifying the strategies, I also looked into a generalisation of the problem, and I found this article about networkBoundResource:

[https://blog.devgenius.io/android-networking-and-database-caching-in-2020-mvvm-retrofit-room-flow-35b4f897d46a][networkBoundResource]

It is a generic function to execute a network request with caching, also available as the library called Flower.

By nature, it's an opinionated function, and it doesn't work for me out of the box, so I wrote my own extensions over it. I do not think that general libraries will work for everyone, so I recommend copying that code into your own projects or library and modifying it according to your heart's desire.

### Monads and other terminology

Let's define a few terms before we delve into the details, so you understand my opinionated solution and can build your own. Some of this is arbitrarily defined by me; others were defined by the libraries we use.

**Result** classes are monads, usually containing a Success and a Failure part, like in the Kotlin standard library. Some people, like the writer of the above library, prefer to add a Loading state to the mix, but I do not recommend this. The loading state is a presentation layer concern, while the monad should be created in the repository layer. Let's not couple the two more than required. I do not use the standard Result class, however. I use the [kotlin-result][kotlin result] library, which is very similar but contains a lot of useful extensions.

The **Response** class is a Retrofit class that represents the network response.

The **Reply** classes with the suffix Reply represent network classes (Data Transfer Objects) that contain business objects PLUS metadata or wrapping, so we have to extract information from them instead of simply mapping them. I introduced this terminology to make the distinction between these and simpler mappings. This is useful in some procedural situations, like templating. The metadata can be paging data or a wrapper around the business data, for example, a ***result*** object in the JSON response. This name would be obviously confusing to be used in the domain or presentation layer, so we strip it out or rename it to Reply.

There are other terms we could reserve for other purposes, like **Hit** for cache hits, **Event** for user events, and **Answer** for classes returned from functions, but I do not use them extensively at the moment, so I won't explain them in detail. Let's reserve these for future use.

## My take on the generic functions

Another problem with the Flower library above is that it hardcodes the strategy, or, to put it another way, it doesn't provide solutions for all the strategies I outlined above. Indeed, you do not need generic functions for the network only and for the cache only strategies, but I would like to use something for the cache first strategies. For this reason, I created two files: ***fetchNetworkFirst*** for the case covered by Flower, and ***fetchCacheThenNetwork*** for the *'Cache First'* cases. The reason we can cover all three cases in the latter is that they differ only in minor details, so we can introduce the strategies as parameters.

All right, it's time to see some code. The rest you can check out [here] [getresult]. You can also have a look at the tests I wrote for these functions [here] [getresult-test].

```kotlin

inline fun <REMOTE, DOMAIN> fetchCacheThenNetwork(
    crossinline fetchFromLocal: () -> Flow<DOMAIN?>,
    crossinline shouldMakeNetworkRequest: (DOMAIN?) -> Boolean = { true },
    crossinline makeNetworkRequest: suspend () -> REMOTE,
    noinline saveResponseData: suspend (REMOTE) -> Unit = { },
    crossinline mapper: REMOTE.() -> DOMAIN,
    strategy: STRATEGY = CACHE_FIRST_NETWORK_SECOND,
) = flow<ApiResult<DOMAIN>> {

    val localData = fetchFromLocal().first()
    localData?.let { emit(Ok(it)) }
    val networkOnlyOnceAndAlreadyCached = strategy == CACHE_FIRST_NETWORK_ONCE && localData != null
    if (shouldMakeNetworkRequest(localData) && networkOnlyOnceAndAlreadyCached.not()) {
        val result = apiRunCatching {
            makeNetworkRequest()
        }.andThen { dto ->
            saveResponseData(dto)
            Ok(dto)
        }.map {
            it.mapper()
        }.recoverIf(
            { _ -> localData != null },
            { null }
        ).mapError {
            it
        }
        if (shouldEmitNetworkResult(result, strategy, localData == null)) {
            emitAll(
                flowOf(
                    result.map { it!! }
                )
            )
        }
    }
}

fun <DOMAIN> shouldEmitNetworkResult(
	result: Result<DOMAIN?, Throwable>,
	strategy: STRATEGY,
	isLocalNull: Boolean
): Boolean {
	return when (strategy) {
		CACHE_FIRST_NETWORK_LATER -> isLocalNull
		else -> result is Err || (result is Ok && result.component1() != null)
	}
}

```
{: file='fetchCacheThenNetwork.kt'}

The above is the version that covers the three Cache First strategies. Notice that the strategy will help to determine the behaviour around ***networkOnlyOnceAndAlreadyCached*** (whether we have to do the network call at all) and ***shouldEmitNetworkResult***.

The ***fetchFromLocal*** function is accessing the cache. It will return a domain object, as the database is responsible for the mapping. The Flower library deals with the database mapping as well, so it needs a DB generic parameter too, but I found this redundant.

The ***makeNetworkRequest*** function will return a REMOTE object because the Retrofit interface won't do the mapping for me. Hence we need the ***mapper*** function, which, for me, is always an extension on the REMOTE class. Earlier versions of this did not have the mapper but saved and loaded data in the database, which also executed the mapping part. I decided to explicitly add the mapper here to avoid relying on the database, having single responsibility, and because I might want to do the database save in a different thread, so I wouldn't wait for the result.

Note that ***apiRunCatching*** is just my extension over ***runCatching*** to avoid import clashes with the Kotlin standard library and make the code more readable. I also use a typealias (***ApiResult***) for ***Result*** for the same reason.

> Use runCatching very carefully. It catches everything, including CancellationException and Error, which is a mistake. You need to rethrow these, or make other adjustments. I will write another article about error handling.<br>
{: .prompt-danger }

The ***saveResponseData*** part will save the data and pass the REMOTE object as an unchanged Result. The mapper will map it to a DOMAIN object, after which the recoverIf function will map the error to a null Result if we have local data already. All these are done with the help of kotlin-result extension functions.

There is a separate ***Response*** version for each function in the same file because they will take different parameters wrapped in Response classes, recover from additional scenarios (like the eTag not changed exception), and have different ways to extract information from headers and error bodies.

In the ***mapError*** part, you can invoke your app-wide error handler to map the throwables to domain-specific exceptions. You could modify this by adding the mapping function as a parameter. I simply pass on the throwable as it is for now.

Please note that this solution uses cold flows and expects a single database source, so if you want to separately fire requests and then passively listen to hot flows from the database, or if you want to include in-memory cache in the process, you might have to introduce big changes to it. Also, it doesn't allow combining multiple calls into one or paging for now. There is still a lot to improve and experiment with here. Let me know in the comments how you would improve it.

## Conclusion

That was a pleasureful journey for me, and I hope you enjoyed it as well, dear reader. I still have a lot to learn and then, hopefully, share. I might venture into ***in-memory caches*** or ***benchmarking*** the various strategies. But for now, I will move on to other things, as there are so many matters to explore in the Android world.


[caching-strategies]: https://borstch.com/blog/caching-strategies-in-pwa-cache-first-network-first-stale-while-revalidate-etc
[mastering-caching-strategies]: https://thinhdanggroup.github.io/caching-stategies/
[universal-caching]: https://medium.com/@andrei_r/easy-caching-android-kotlin-flow-b824a29e8a77
[rfc5861]: https://datatracker.ietf.org/doc/html/rfc5861
[networkBoundResource]: https://blog.devgenius.io/android-networking-and-database-caching-in-2020-mvvm-retrofit-room-flow-35b4f897d46a
[kotlin result]: https://github.com/michaelbull/kotlin-result
[getresult]: https://github.com/herrbert74/FlickSlate/tree/main/app/src/main/java/com/zsoltbertalan/flickslate/util/getresult
[getresult-test]: https://github.com/herrbert74/FlickSlate/tree/main/app/src/test/java/com/zsoltbertalan/flickslate/util/getresult
