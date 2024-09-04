---
layout: post
title:  "Architecture related decisions in Android - Mapping"
date:   2024-07-17 17:10:00 +0000
categories: android architecture
mermaid: true
---

![starting-image](/assets/img/posts/20240717_architecture_mapping.jpg){: width="512" height="512" }

<a href="https://androidweekly.net/issues/issue-632"><img alt="Featured in Android Weekly Issue 632" src="/assets/img/posts/20240717_badge.svg" width="252" height="20"></a>

Other parts in the series:<br>
[Architecture related decisions in Android - Introduction]<br>
[Architecture related decisions in Android - Error handling and Monads]<br>
Architecture related decisions in Android - Mapping (this article)<br>
[Architecture related decisions in Android - Response and Reply classes]<br>
[Architecture related decisions in Android - The rest]<br>

## Why map? Cannot we just use the same model everywhere?

In order to separate concerns between the layers of your architecture, you will need to have separate **models** for each layer. Your **network** model will contain nonidiomatic naming, unwanted fields, renaming annotations, and malformed data. Your **database** model might use annotations that you don't plan to use in your UI, and so on. And mapping is the place to create the perfect boundaries between your layers.

You can get away with it for a long time. But once you realise how much you are better off, there is no way back. For me, that moment came a few years ago, when I was forced to use GraphQL at work. This made it necessary to create a separate network and domain/presentation model.

There is just no better explanation on this topic than [this][do-you-even-map-though] article, which was written at a time when we all started to feel the need for something like this.

```mermaid
---
title: Mapping directions
---
flowchart TD
    domain(Domain model)
    network(Network model)
    database(Database model)
    presentation(Presentation model)
    network-->domain
    domain-->database
    domain-->presentation
    domain-->network
    database-->domain
    presentation-->domain
```

## A note on mapping the presentation layer model

Until recently, it didn't even occur to me to map between the **presentation** and **domain** models. I just used the domain model with Android dependencies if needed. A short while ago, I received feedback on a test challenge that I hadn't separated the presentation layer model from the domain one. I really felt that this was premature optimisation, because the two layers didn't have any differences.

While I researched for this post, however, every single article about mapping recommended this separation. Like the one I linked above, [this][how-to-map-data-between-layers] is a great article too.

But I still maintain that you don't really need to do this in every single case. Something to consider is that even if the domain has some Android dependencies, is it worth it to separate them? Could you not test the domain layer then? Is there any other downside to that? If the answer is no, do not create a separate model layer at all.

## Do we need to test mapping?

Fairly recently, I received feedback from yet another test challenge. One of the questionable points was that I added **tests for mapping**. They said these tests are meaningless, and I should have focused on the business logic.

My first trouble with this is that sometimes there is barely any business logic to test, so by default I start with adding tests for mapping. Sometimes this also helps me understand the code and the problem better.

The other one is that mapping is the instrument to separate concerns between the layers of the application. If there is some trouble, I want to know about it as early as possible.

So I don't think these tests are completely useless. Therefore, increasing the code coverage with them is not completely impractical.

## Where to do the mapping?

This is the most relevant question to this article series, but I couldn't find a solution that is consistent with everything else I wrote.

The problem is that for complete separation of concerns, you should do the mapping on the edges of the application, meaning in the presentation, data (network, and database) layers, not in the domain layer. This is fine for database and presentation (if we use it), but not so clear in the network layer. The reason is that for Retrofit, we usually generate implementation with a Builder, and we simply use that in the **Repository**, meaning we map in the Repo implementation, which is close to the data layer, but it's not consistent with the mappings for the database and presentation.

One possible solution is in [this article][modeling-retrofit-responses-with-sealed-classes-and-coroutines], in the form of custom **CallAdapters**. This helps to move the mapping to the correct layer, but at the cost of a much more complex and unclear mapping, which is now inconsistent with the mapping in the other layers.

After some consideration, I decided for myself that it's just not worth it for me, but you might have a different opinion, which is probably valid as well. As you can see, I highly value **consistency**, but sometimes I prize **convenience** or **simplicity** even more.

## Mapper functions and naming

There is no need to create mapper objects and inject them anymore. The dependent class can be used as the location for an **extension function** mapper. Because we always map between the domain and something else (network, database, or presentation layers), and the domain is always the dependency, we can put the mapping next to the other classes.

The receiver can always be the source class that we map from. The target is the other class. This way, we avoid functions named like **fromSomething()**. This means that if we need back and forth mapping, like for the database classes, we will have **Something.toDbo()** and **SomethingDbo.toSomething()**, and not *SomethingDbo.fromDbo() : Something*.

Please note that you might prefer **toDomain()** instead of toSomething(). That's just my preference, which, I admit, is somewhat illogical.

This is how it looks:

```kotlin
fun CategoryDto.toCategory() = Category(
	this.id ?: 0,
        ...,
	this.type ?: "",
)

fun Category.toDto() = CategoryDto(
	this.id ?: 0,
        ...,
	this.type ?: "",
)
```
{: file='CategoryDto.kt'}

```kotlin
fun CategoryDbo.toCategory() = Category(
	this.id ?: 0,
        ...,
	this.type ?: "",
)

fun Category.toDbo() = CategoryDbo(
	this.id ?: 0,
        ...,
	this.type ?: "",
)
```
{: file='CategoryDbo.kt'}

Please also note that **toSomething()** and **asSomething()** functions have a different meaning in Kotlin. The toSomething() function will return a new object, just what we need. The asSomething() functions are designated for collection mapping, where we maintain a reference between them. That's not our use case here.

## Nullability

Mappers are the best place to remove unwanted nullability. It's often the case that we need to define the DTO classes defensively as nullable, but this is undesirable in the domain layer and beyond. So we execute the mapping by adding sensible defaults. In most cases, these are never actually needed, just to satisfy the compiler. The null values in these cases mean an error, but as we wrapped our value in Result classes, these empty default values will never be used.

**Unless** you use a completely different approach from mine and you use null values to signify errors, which I do not recommend.

**Or** the values are actually nullable in the domain, of course, which is sometimes the case. 

## Mapping lists

There are some very useful extension functions that can be used to simplify the **mapping of complicated lists**.

```kotlin
// Non-nullable to Non-nullable
inline fun <I, O> mapList(input: List<I>, mapSingle: (I) -> O): List<O> {
	return input.map { mapSingle(it) }
}

// Nullable to Non-nullable
inline fun <I, O> mapNullInputList(input: List<I>?, mapSingle: (I) -> O): List<O> {
	return input?.map { mapSingle(it) } ?: emptyList()
}

// Non-nullable to Nullable
inline fun <I, O> mapNullOutputList(input: List<I>, mapSingle: (I) -> O): List<O>? {
	return if (input.isEmpty()) null else input.map { mapSingle(it) }
}
```
{: file='ListMappers.kt'}

The idea is that you don't have to repeat the same list mapping over and over. The above functions will reduce the amount of work needed with the help of an inlined generic function.

And the usage:

```kotlin
fun List<CategoryDto>.toCategoryList(): List<Category> = 
    mapNullInputList(this) { categoryDto -> categoryDto.toCategory() }

fun CategoryDto.toCategory() = Category(
	this.id ?: 0,
        ...,
	this.type ?: "",
)
```
{: file='CategoryDto.kt'}

## Conclusion

And that's it. If you haven't started mapping yet, it's high time to do it. And if you have, but you have a different opinion, please let me know in the comments. Happy mapping!

[Architecture related decisions in Android - Introduction]: https://herrbert74.github.io/posts/architecture-related-decisions-introduction/
[Architecture related decisions in Android - Error handling and Monads]: https://herrbert74.github.io/posts/architecture-related-decisions-error-handling-and-monads/
[Architecture related decisions in Android - Response and Reply classes]: https://herrbert74.github.io/posts/architecture-related-decisions-response-classes/
[Architecture related decisions in Android - The rest]: https://herrbert74.github.io/posts/architecture-related-decisions-rest/
[how-to-map-data-between-layers]: https://proandroiddev.com/app-architecture-how-to-map-data-between-layers-df0179c52f04
[do-you-even-map-though]: https://buffer.com/resources/even-map-though-data-model-mapping-android-apps/
[modeling-retrofit-responses-with-sealed-classes-and-coroutines]: https://proandroiddev.com/modeling-retrofit-responses-with-sealed-classes-and-coroutines-9d6302077dfe
