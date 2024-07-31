---
layout: post
title:  "Architecture related decisions in Android - Introduction"
date:   2024-06-27 10:57:00 +0000
categories: android architecture
---

![starting-image](/assets/img/posts/20240627_architecture.jpg){: width="512" height="512" }

Other parts in the series:<br>
Architecture related decisions in Android - Introduction (this article)<br>
[Architecture related decisions in Android - Error handling and Monads]<br>
[Architecture related decisions in Android - Mapping]<br>
[Architecture related decisions in Android - Response and Reply classes]<br>
*Architecture related decisions in Android - The rest* (coming soon)

## What is architecture?

Architecture is the way various components on different layers of an application relate to each other. It could be high-level, such as UI, model, and data relations, but it can be more focused, like MVP, MVVM, and MVI architectures in the presentation layer.

When talking about architecture, a lot of Android developers use MV* architecture interchangeably with app architecture, which are two largely different things. While researching for this article, I ran into a few articles where the author could not even understand this distinction.

In this article series, I wanted to talk only about smaller decisions as they influence other parts of the architecture, but due to the above reasons, in this introductory article, I will talk about app architecture and presentation layer architecture.

In the rest of the series, I will write about decisions that will influence several parts of the architecture, in particular:
* Result Monad and Error handling
* Mapping
* Response classes
* Finally, some smaller decisions

As always, I researched the internet for related articles. If there is a good one, I don't want to steal or repeat the information in it. I will link to all of these blog posts, commenting on them and expanding on them as necessary. I hope this will mean for a lot of you to go down a lot of rabbit holes and read not only these five articles but much more.

## Google recommendations

For a start or refresher, I recommend reading [Google's Guide to app architecture][architecture-guide]. While in the past their recommendations were lacking or straight wrong, now I consider them a good starting point.
In this article, I also want to show you where you might want to consider moving away from their recommendations.

## Alternatives to Google architecture recommendations

They also have a more detailed [architecture recommendation][architecture-recommendations]. I recommend familiarising yourself with this too and following most of it. But I would like to provide some alternatives here as well.

### ViewModels

You can even use ViewModels in Kotlin Multiplatform now. Still, for Kotlin Multiplatform, I recommend using [Decompose][decompose] and [MVIKotlin][mvikotlin]. They provide better separation of concerns, better navigation, better dependency inversion, and a pluggable UI. For details, see the links.

### Hilt

I strongly recommend using Dagger and Hilt. For Kotlin Multiplatform, you are forced to use something else, at least at the time of writing. I recommend [Koin][koin] or [kotlin-inject][kotlin-inject]. For my [Kotlin Multiplatform project][companies-house], I settled on Koin. I'm not sure if it was the best decision, but it was a solid one.

## Clean Architecture

The Google recommendation in the above section is Clean(ish) Architecture.

### What is Clean Architecture?

Clean Architecture pushes us to separate stable business rules (higher-level abstractions) from volatile technical details (lower-level details), defining clear boundaries.
[Here][clean-architecture-gist] is a quick summary if you need one.

The main difference between the Google recommendation and true Clean Architecture, as I understand it, are Use Cases. Hence, the next section.

### Use Cases

The Use Case encapsulates the business logic for a single reusable task the system must perform.

### When to use Use Cases?

* When the Use Case needs to be reused
* When we need to aggregate results from more than one repository

Consistency is important, but in a lot of situations, you have to introduce some flexibility. Use Cases are a good example of that. A consistent approach would be to use Use Cases all the time or never. Both suffer from inflexibility, in my opinion. Use Cases in simple CRUD situations would bloat the code base, while never using them would hurt reusability and separation of concerns.

An example when you do not need flexibility is the role of the Repository. A lot of developers (including a younger myself) use repositories to directly access data sources. I recommend always using an abstraction over your data sources, for example, a BooksApi interface for the network and a BooksDataSource interface for the database.

Unlike the presentation layer and repositories, Use Cases should not start coroutines or switch threads. They are suspend functions, which call suspend functions, or normal functions calling funtions returning flows. This is why they are optional: if they have a single usage on a single repository, they are simple call-through, useless code.

I recommend reading this article, which also proposes flexibility and goes into more details:<br>
[https://medium.com/@VolodymyrSch/the-complexities-of-clean-architecture-use-cases-71ac89ea8b40][use-cases]

Some people seem to advocate using either Use Cases or repositories. They are not interchangeable. I defined roles for Use Cases earlier. Repositories aggregate various data layers, like network and database. They are also responsible for offloading work to other threads and returning results to the calling thread.

> Verdict: Use Clean(ish) Architecture. I recommend Use Cases, but only when they make sense.
{: .prompt-warning }

## Presentation layer architecture

There are only two options I would consider: MVVM and MVI. I won't go into details with them; there are a lot of articles on the internet already. They are very similar; the only difference between them is that MVI has a single state object and that MVI has a well-defined intent structure.

For MVI, I recommend Decompose and MVIKotlin, which I already mentioned above.

A word of warning when you choose an architecture for interview take-home test challenges: a lot of reviewers won't understand or simply don't want to consider MVI at all, and your test will fail straight away. Others will find your choice unjustified or will misunderstand or miss some constructs. For this reason, for take-home challenges, I recommend using MVVM. It's useful to know the two main architectures anyway.

> Verdict: Use either MVVM or MVI. Either use libraries or roll your own. Use MVVM for take home test challenges.
{: .prompt-warning }

[architecture-guide]: https://developer.android.com/topic/architecture
[architecture-recommendations]: https://developer.android.com/topic/architecture/recommendations
[decompose]: https://arkivanov.github.io/Decompose/
[mvikotlin]: https://arkivanov.github.io/MVIKotlin/
[koin]: https://insert-koin.io/
[kotlin-inject]: https://github.com/evant/kotlin-inject
[companies-house]: https://bitbucket.org/babestudios/companies-house
[clean-architecture-gist]: https://gist.github.com/ygrenzinger/14812a56b9221c9feca0b3621518635b
[use-cases]: https://medium.com/@VolodymyrSch/the-complexities-of-clean-architecture-use-cases-71ac89ea8b40
[Architecture related decisions in Android - Error handling and Monads]: https://herrbert74.github.io/posts/architecture-related-decisions-error-handling-and-monads/
[Architecture related decisions in Android - Mapping]: https://herrbert74.github.io/posts/architecture-related-decisions-mapping/
[Architecture related decisions in Android - Response and Reply classes]: https://herrbert74.github.io/posts/architecture-related-decisions-response-classes/
[Architecture related decisions in Android - The rest]: http://localhost:4000/posts/architecture-related-decisions-rest/