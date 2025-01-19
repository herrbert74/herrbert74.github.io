---
layout: post
title:  "Android Architecture - Data Layer"
date:   2025-01-19 09:10:00 +0000
categories: android architecture
mermaid: true
---

![starting-image](/assets/img/posts/20241219_data_layer.jpg){: width="800" height="535" }

I see a lot of confusion in apps around data layers. What is a **Data Source**? What is a **Repository**? What is an **Interactor**? Different people will have different definitions and use them interchangeably. In my opinion these are indeed somewhat convertible, but each has a core functionality.

This confusion prompted me to look into that more deeply, and it resulted in a more clear view of the subject. It even made me change the architecture of my experimentation app, which from now on will have four layers for data architecture.

In this article I try to summarise my definitions, with clarity and separation of concerns in mind first and foremost.

The above mentined four layers are:

### Data service layer

The only mandatory layer, because this contains the actual data services, like **Retrofit** and **Room** interfaces and their implementations. The output is always **DTO** (Data Transfer Object) and **DBO** (Database Object or Entity) classes.

### Data Source layer (optional)

This layer separates the data service implementations from the layers above it. 

* Its main function is to do the **threading** when the data service layer is not doing it. Please refer to my article:
  * [https://herrbert74.github.io/posts/architecture-related-decisions-rest/#where-to-use-concurrency][where-to-use-concurrency]
* Another important function of this layer is to **convert between DTOs or DBOs and domain classes**. This includes extracting information from implementation details like Retrofit Responses and Response headers.
* When the Data Source layer is omitted, both the service layer and the Repository layer could take on the converting job. Another frequently used option is to use separate Converter classes, but I find them superfluous.

### Repository layer (optional)

* This layer is normally only used to execute a **caching strategy**, **aggregating results** from network calls and the database or caching layer.
* Also responsible for **retry** attempts and **recovery** after recoverable exceptions.
* When the Data Source layer is omitted, it can be used to **convert between different data layers** and **hide the data services** from the domain.
* When the Interactor layer is omitted, it could be used to **aggregate different repositories or services**.

As you can see, this layer can get really complicated. For a generic, inlined solution that covers everything above, check out my fetcher functions:

[https://herrbert74.github.io/posts/caching-strategies-in-android#getresults-a-collection-of-repository-functions][caching-strategies-getResults]

Please note that the fetcher functions have changed since my article about them. I will write a follow-up article to introduce the changes.

For now, you can take a look at the relevant part in my project:
[https://github.com/herrbert74/FlickSlate/tree/main/shared-data/src/main/java/com/zsoltbertalan/flickslate/shared/data/getresult][getResult]
 
### Interactor layer (optional)

The interactor layer is **not part of the data layer but the domain layer**, but it optionally connects the rest of the application to the data layer, so I want to discuss it a little here.

The interactor layer encodes the real-world business rules that determine how data can be created, stored, and changed. Interactors are the **technical implementations** of the more user-centric **use cases**. So it's important to understand that the two concepts are essentially the same; they are the same stuff viewed from different angles.

Both the presentation and the data layer can get really complicated and duplicated. The domain layer may introduce interactors to mitigate this. **They can encompass duplicated functions from ViewModels or can use several repositories** that are used in more than one place. This is the reason I mention interactors as part of the data layer.

No threading should be done in this layer. No Android dependencies and data layer implementations should serve as dependencies for this layer.

# Clean architecture

Let's see how a clean architecture diagram would look based on the above information.


```mermaid
block-beta
  columns 1
  a["data-service"]
  space
  b["data-datasource"]
  space
  c["data-repository"]
  space
  d["interactor"]
  space
  f["ui-viewmodel"] 
  space
  g["ui-view"]
  b-->a
  c-->b
  d-->c
  f-->d
  g-->f
  classDef planned fill:#6f6
  class b planned
  class c planned
  class d planned
  class e planned
```

It's worth mentioning that my interpretation does not suggest that the domain layer depends on the data layer. **The interactor will depend on the repository interface**, which sits in the domain layer. As a result, **my suggestion above is in the middle of the original Clean Architecture and Google's Cleanish Architecture**. Google recommends adding the repository interfaces to the data layer and thus reverting the dependency between the domain and the data layer. So far I've been averse to this idea, despite the added simplicity.

There is an interesting [discussion][nowinandroid-ca-discussion] about the above in the NowInAndroid GitHub repository. To summarise, both approaches (including mine) are valid, and your decision is mostly down to personal preferences, project size, project structure, complexity, and future projections of them.

# Variations based on project size

As you can see, for different requirements you will have to use different layers as needed. The Data Source and Repository layers are easy to confuse, and they are often used interchangeably. I would recommend omitting the Data Source layer before omitting the Repository layer, so not using several services or aggregate Data Source in the Data Source themselves.

This is my **current** take on this dilemma:

### Minimum Viable Project (MVP) project likely to be replaced

You might only need the Service layer and nothing else. An optional Repository layer could be added to separate concerns, which I prefer even in the simplest situations.

### MVP project likely to be continued

Like the above, but you might want to consider if various features will benefit from a more complex structure even if it's not immediately needed.

### Medium sized project

I would add Data Sources and Repository by default. Customise the feature as needed by removing Data Source or adding Interactors as needed.

### Large project

You will probably need all layers across the app, but when you separate your features, you have to use your judgement to omit layers depending on the size and needs of your feature.

# Conclusion

The article discusses the confusion surrounding data layers in apps, focusing on four layers: **Data service layer, Data Source layer, Repository layer, and Interactor layer**. The **Data Service layer** implements data services, while the **Data Source layer** separates implementations and converts between DTOs and domain classes. The **Repository layer** executes caching strategies and handles retry attempts and recovery. The **Interactor layer** aggregates different repositories or services. I suggest omitting the Interactor layer before the Data Source layer and in turn that before the Repository layer.

[where-to-use-concurrency]: https://herrbert74.github.io/posts/architecture-related-decisions-rest/#where-to-use-concurrency
[caching-strategies-getResults]: https://herrbert74.github.io/posts/caching-strategies-in-android#getresults-a-collection-of-repository-functions
[getResult]: https://github.com/herrbert74/FlickSlate/tree/main/shared-data/src/main/java/com/zsoltbertalan/flickslate/shared/data/getresult
[nowinandroid-ca-discussion]: https://github.com/android/nowinandroid/discussions/1273