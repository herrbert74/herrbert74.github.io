---
layout: post
title:  "Architectural Evolution of and Android app"
date:   2025-08-18 17:50:00 +0000
categories: android architecture modularisation
mermaid: true
---

![starting-image](/assets/img/posts/20250818_arch_evolution.jpg){: width="1024" height="576" }

This article will explain the stages you will probably go through as your Android application grows. Different architectures might look vastly different (not to mention different opinions on details), but the thinking behind them remains the same. I will explain this through Clean Architecture and my [FlickSlate][FlickSlate] repository.

I also want to introduce common patterns in large applications: **super layers** and **feature groups**. These came up every time when the apps I worked on reached a certain size, but I never formulated the basic rules around it. I hope writing them down will make it easier to understand for me and others as well.

### Super Layers

As you might remember from my previous articles, my apps have an independent **domain layer**, on which the **data and presentation (from now on UI) layers** depend. These are the basic horizontal layers. The verticals containing them are also called features.

I created the term **'super layers'** to introduce a broader hierarchy over the basic horizontal layers. We need them because the scope of horizontal layers might be different, meaning they can span not only features, but also something wider. I will define **feature**, **feature group**, **app-wide (shared)**, and **multi-app (base)** super layers.


Please refer to the diagram below.


```mermaid
block-beta
  columns 8
  block:features:6
    fa["Feature A"] fb["Feature B"] fc["Feature C"] fd["Feature D"] fe["Feature E"] ff["Feature F"]
  end
  space:2
  block:featuregroups:6
    g1["Feature Group 1"]
    g2["Feature Group 2"]
  end
  space:2
  Shared:6
  space:2
  b1:6 space b2
  b1["Base"]:6-->b2["Same in another app"]
```

In the basic case I explained above, we have the **feature layer**, where clean architecture layers are only visible within the feature.

Next, the **feature group** layer is an intermediate layer between application-scoped and feature-scoped layers. I will discuss this later in detail, as this is the last and least obvious addition to the super layers.

A **shared, common or app-wide layer** typically contains business-related or unique classes for the whole application. This is the lowest layer where domain, data, and UI modules can be added.

A **base, infrastructure, or foundation layer** could be used across multiple applications. It contains no domain, data, or UI layers, but classes that are common across these layers. I typically have a **Kotlin and an Android base** module. I created the [BaBeStudios-Base][BaBeStudios-Base] library project to reuse these in several apps, but I also have base modules in my apps for code not in this library project yet. BaBeStudios-Base is also available from [MavenCentral][BaBeStudios-Base-Maven], but documentation doesn't exist yet.

As your application grows and you introduce more modules, you will need to introduce the above layers.

Next, let's see the modularisation steps I usually take as my apps grow.

### Step 1: Single module

When you know that your app won't grow above a **maximum size**, or when you are time-pressed, you go with a single-module application or monolith. I'm not sure about the maximum size for single-module apps. This will probably be different for different people.

This maximum size is also different from the **threshold size, where you should start modularising** your app or, more generally, where you should switch to the next step in the process. I think the threshold size is way smaller than the maximum size of a single module application, meaning you should start to modularise a lot earlier than reaching that limit.

For most applications I even recommend **starting modularised** instead of with a single module, because that has a much smaller overhead and level of disruption than starting to modularise when you are nearing your limit, let's say 10 or 20 kLOC.

This is a package tree example of a single module application.

```

├── data
│   ├── database
│   ├── local
│   ├── repository
│   └── remote
├── domain
│   ├── api
│   ├── model
│   └── usecase
├── presentation
│   ├── master
│   └── detail
└── shared

```

The leaves in the above represent packages, which can contain further packages not shown here. Below I will show a similar structure for modules, where a name starting with a colon represents a module and double colons mean a root module.

### Step 2: Simple Modularisation

For **smaller applications** I use simple modularisation as opposed to full modularisation in the next chapter. Here we introduce **modules for every feature** only. Each feature will contain a domain module, plus data and presentation modules depending on the domain module in my case.

We also introduce a **shared module** containing all the common code for the feature modules.

The super layers of the application are separated by a **horizontal line**. Every layer depends on all the layers beneath it. Data and presentation depend on domain, but this is not shown in the diagram.

```

::feature:Feature A

├── :feature/:featurea/:data
├── :feature/:featurea/:domain
└── :feature/:featurea/:presentation

::feature:Feature B

├── :feature/:featureb/:data
├── :feature/:featureb/:domain
└── :feature/:featureb/:presentation

::feature:Feature C

├── :feature/:featurec/:data
├── :feature/:featurec/:domain
└── :feature/:featurec/:presentation

------------------------------------------

:shared

```
{: file='Simple modularisation'}

### Step 3: Full Modularisation

By full modularisation I mean that after the features, the shared layers get their own modules as well. Usually this means not three, but five modules. Apart from domain, data and presentation, the common Android and Kotlin infrastructure get a module each.

Shared model classes, repository interfaces, and use cases go into **shared/domain**; shared DTOs and DataSources go into **shared/data**; common views or composables go into **shared/ui**. Commonly used Android code, like resources, goes into **base/android**, while commonly used Kotlin code, like extension functions, goes into **base/kotlin**.

Base libraries should be independent of each other, so they should be on the same layer. I added them to separate layers for practical reasons here. If you use Hilt, like me, you either don't use it in **base:kotlin**, or **base:kotlin** will be an Android library, because Hilt is for Android only, and it doesn't mix with pure Dagger.

My **current** solution is to make base:android depend on base:kotlin and make it add the base:kotlin classes to the dependency graph. There is no perfect solution here.

```

::feature:Feature A

├── :data
├── :domain
└── :presentation

::feature:Feature B

├── :data
├── :domain
└── :presentation

::feature:Feature C

├── :data
├── :domain
└── :presentation

------------------------------------------

::shared:data

├── database
├── local
├── repository
└── remote

::shared:domain

├── api
├── model
└── usecase

::shared:presentation

├── compose
├── design
└── util

------------------------------------------

::base:android

------------------------------------------

::base:kotlin

```
{: file='Full modularisation'}

### Step 4: Feature groups

When your app is fully modularised, you get to a point when your shared layers take over the role of a monolith and become **bottlenecks** during the build process.

I consistently run into this above a certain application size, somewhere between 50 and 80 kLOC. You have to share too many things, so your **shared domain and presentation modules are becoming too big**. You start to notice that sharing code among all the verticals is also wasteful, because you only want to share it between two or at most three or four modules. You find that a larger and larger part of new code goes into these modules, and the **incremental caches are less and less effective** because you invalidate them frequently with your changes.

You might say this could be avoided by properly refactoring and structuring the application or by duplicating some code, but this might be harder than you think. I found these not helpful enough. Or you might have a legacy app at hand, where you don't have time to refactor everything perfectly.

My current solution to this problem is identifying **feature groups**, a group of features with a lot of common dependencies. This way you can split some of the code in your shared layers horizontally by creating **more cohesive** modules.

There are two distinct feature groups in all the large applications I worked on recently.

The first one is related to the **core business** features: for example, one module is around a list of products, another around the product details, and a third about favourited products. So we could name this group **Product**.
Another group is around **payments, ads or subscriptions**, or sometimes a combination of these. This is around the way the app is monetised. So we could have a group called **'Payments'** for example.

In large apps there could be five or more groups. Each group will have domain, data, and presentation modules, if needed.

```

feature/Feature A

feature/Feature B

feature/Feature C

feature/Feature D

feature/Feature E

feature/Feature F

------------------------------------------

group/GroupABC aka group/Product

├── :data
├── :domain
└── :presentation

group/GroupDEF aka group/Payments

├── :data
├── :domain
└── :presentation

------------------------------------------

:data

:domain

:presentation

------------------------------------------

:android-base

------------------------------------------

:kotlin-base

```

### When to use which step?

You definitely want to use a monolith with apps that will never go to production or single-use apps. These can be a **Proof of Concept** (PoC) app, a **Minimum Reproducible Example** (MRE) app for third-party bugs, or a **test challenge** app for job applications.

For all others I recommend starting with partial or full modularisation and upgrading to the next step as needed, or slightly earlier if possible, to reduce the pain later.

# Conclusion

You can find a working example of (most of) the above in my [FlickSlate][FlickSlate] repository, which I recently updated to the full modularisation stage. This was premature but useful to demonstrate the principles. And prevent the pain later, of course.

FlickSlate is not upgraded to feature groups yet. It would make no sense, as the shared layer couldn't be split into feature groups yet. There is no monetisation in the app, so only the shows could be grouped, but this is done perfectly by the shared layer for now.

This article mostly comes from the frustration that there is only blanket advice on the internet about when and how to modularise: either it is recommended to do it indiscriminately, or somebody is shouting 'overengineering' at anyone who is doing it. I hope this article helped to make more informed decisions on the when and also on the how.

[BaBeStudios-Base]: https://bitbucket.org/babestudios/babestudiosbase/src/master/
[BaBeStudios-Base-Maven]: https://mvnrepository.com/artifact/io.bitbucket.babestudios
[FlickSlate]: https://github.com/herrbert74/FlickSlate