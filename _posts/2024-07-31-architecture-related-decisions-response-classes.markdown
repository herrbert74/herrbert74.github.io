---
layout: post
title:  "Architecture related decisions in Android - Response classes"
date:   2024-07-31 11:10:00 +0000
categories: android architecture
---

![starting-image](/assets/img/posts/20240731_response.jpg){: width="512" height="512" }

The words **Result**, **Response**, **Reply**, and **Answer** are synonyms in normal everyday conversations. They can be used interchangeably. However, we software engineers like to add extra meaning to words in order to increase clarity and consistency in our code. There are **industry-wide** accepted added meanings, like Result, **langauge-specific**, like Response, and **company level** or **individually** used, as in the case of Reply and Answer.

Other parts in the series:<br>
[Architecture related decisions in Android - Introduction]<br>
[Architecture related decisions in Android - Error handling and Monads]<br>
[Architecture related decisions in Android - Mapping]<br>
Architecture related decisions in Android - Response and Reply classes (this article)<br>
[Architecture related decisions in Android - The rest]<br>

## Response vs. Reply

**Response** classes are commonly used and easily understood, so let's start with these. They represent a wrapper around business data, which also includes some metadata, like headers. You also have access to the response body and the error body.

I use the term **Reply** to make a distinction from Response. They are also wrapper classes around your business data, but only within the response body. This means that a Response class can wrap a Reply class, but not vice versa. In this article, I will show you some examples to make the difference clear.

## Response

**Response** classes come from libraries like Retrofit and Ktor. My examples will use Retrofit, but it applies similarly to Ktor, and probably other network layer libraries as well. 

They represent a HTTP response. You can have access to the header with metadata, the body, and the error objects through it.

Response class usage is limited to the data layer. I only include them in this article, because they infuence the naming of other similar classes.

### When we should use Response class

If we need only the response body, we can omit the Response class. Let's see when we still need it.

#### We need to extract metadata from header

We can get information from the header, like content type and cache information. I frequently use **ETag**s to reduce network traffic, so let's take this as an example.

**ETag** is a hash code generated by the backend to identify the exact content of a resource. So if it has not changed, we will get an HTTP 304 error code instead of the payload, and we can reuse the cache.

I use this procedure in my FlickSlate app that I mentioned before. Let's look into the **Genre** loading, which you can check out from here if you like:

[https://github.com/herrbert74/FlickSlate/blob/main/app/src/main/java/com/zsoltbertalan/flickslate/data/repository/GenreAccessor.kt][FlickSlate-GenreAccessor]

This is the corresponding Retrofit function. Notice that it is using Response as a wrapper around the Reply class. The **ifNoneMatch** parameter will also be important in the next steps.

```kotlin
@GET(URL_GENRE)
suspend fun getGenres(
    @Query("api_key") apiKey: String = BuildConfig.TMDB_API_KEY,
    @Query("language") language: String? = "en",
    @Header("If-None-Match") ifNoneMatch: String = ""
): Response<GenreReply>
```
{: file='FlickSlateService.kt'}

And this is how we use the above and the ETag. **GenreAccessor** is my Repository implementation. The **fetchCacheThenNetworkResponse** is a useful function, about which I wrote my [Caching Strategies][Caching-Strategies] article.

We pass the **makeNetworkRequest** function into it, where we pass the ETag retrieved from the database into the **ifNoneMatch** parameter, which will be added into the request header.

If the call is successful, we will receive a new ETag, which we save into the database in the **saveResponseData** function.

The fetch function will internally take care of the case where the ETag matches the current response from the remote, so it will return a **HTTP 304** error. In this case, we return the data retrieved from the database.

```kotlin
override fun getGenresList(): Flow<Outcome<List<Genre>>> {
    return fetchCacheThenNetworkResponse(
        fetchFromLocal = { genreDataSource.getGenres() },
        makeNetworkRequest = {
            val etag = genreDataSource.getEtag()
            flickSlateService.getGenres(ifNoneMatch = etag)
        },
        saveResponseData = { genreResponse ->
            val etag = genreResponse.headers()["etag"] ?: ""
            genreDataSource.insertEtag(etag)
            val genres = genreResponse.body()?.toGenres().orEmpty()
            genreDataSource.insertGenres(genres)
        },
        mapper = GenreResponse::toGenres,
    ).flowOn(dispatcher)
}
```
{: file='GenreAccessor.kt'}

#### We need to extract data from the ErrorBody

Sometimes the error message is not in the message field of the Throwable, and you have to extract it from the error body. For example, the TMDB API will return messages like this in the body:

```kotlin
{
    "success": false,
    "status_code": 3,
    "status_message": "Authentication failed: You do not have permissions to access the service."
}
```
{: file='AccountStatus.json'}

In this case, you have to extract the message from the error body in case you need to display it. The good news is that you still **do not have to declare the Response wrapper**, because Retrofit will include it in the **HttpException response** field. You can extract and convert the error body when you catch the error.

In my case, this happens through my runCatchingApi function, which uses runCatching from kotlin-result:

```kotlin

inline fun <V> runCatchingApi(block: () -> V) = runCatching(block)
	.mapError { it.handle() }

```
{: file='KotlinResult.kt'}

The handle function will take care of the rest. There you can extract the message from the Throwable by converting it to a **HttpException** and calling response()?.errorBody() on it.

## Reply

**Reply** classes represent a custom wrapper over your business data, as it comes from the backend in the Response **body**. It usually represents the whole JSON response, where the business data is wrapped in something named ***result***, ***response***, ***data***, ***answer***, ***items***, and many other names, but most of these are already used for something else. Notice that something named response in a json file might be converted to a Reply class, and that's fine!

I came up with the word Reply because everything else seemed to be occupied already. It's a good idea to keep the naming consistent so everybody instantly knows what I'm talking about. At least in the Android team, that is.

You don't necessarily need to add a Reply suffix to the DTO class. Sometimes I prefer to use the business class with a DTO suffix.

The rest of the class is usually some metadata, like paging data, status codes, and messages.

A simple example:

```json

{
    status: "success",
    message: "Success",
    data: {
        user: {
            id: "1"
            name: "Zsolt Bertalan",
            occupation: "Android Developer"
        }
    }
}

```
{: file='user.json'}

This can be translated to a UserReply class:

```kotlin

data class UserReply(
    val status: String,
    val message: String,
    val data: User
)

data `data`(val user: User)
data User(...)

```
{: file='UserReply.kt'}



The Reply class might be generic on the business class. The following is the Order equivalent of the previous User Reply:

```json

{
    status: "success",
    message: "Success",
    data: {
        order: {
            id: "1000000",
            user: "1",
            article: "1",
            amount: 1
        }
    }
}



```
{: file='order.json'}

This might be translated as a Reply class over an Order class:


```kotlin

class Reply<T> {
    var status: String
    var message: String
    var data: T
}

```
{: file='Reply.kt'}


```kotlin

...
val userReply: Reply<User>
val orderReply: Reply<Order>


```
{: file='Usage.kt'}

#### Where to unwrap the Reply class

The reason that sometimes we do not need the Reply class is that we want to unwrap it for the domain classes, so they are mostly restricted to the network layer.

The metadata can be discarded, or if needed, it can be made part of the domain class. The error messages can be converted to a domain-specific Exception class.

## Conclusion

We learned about the differences between Response and Reply classes. I'm curious how you use these. Let me know in the comments.

[Architecture related decisions in Android - Introduction]: https://herrbert74.github.io/posts/architecture-related-decisions-introduction/
[Architecture related decisions in Android - Error handling and Monads]: https://herrbert74.github.io/posts/architecture-related-decisions-error-handling-and-monads/
[Architecture related decisions in Android - Mapping]: https://herrbert74.github.io/posts/architecture-related-decisions-mapping/
[Architecture related decisions in Android - The rest]: https://herrbert74.github.io/posts/architecture-related-decisions-rest/
[FlickSlate-GenreAccessor]: https://github.com/herrbert74/FlickSlate/blob/main/app/src/main/java/com/zsoltbertalan/flickslate/data/repository/GenreAccessor.kt
[Caching-Strategies]: https://herrbert74.github.io/posts/caching-strategies-in-android/
