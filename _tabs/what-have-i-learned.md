---
layout: page
title: What have I learned?
permalink: /what-have-i-learned/
icon: fas fa-leaf
order: 8
---

## 2024

### August

* Custom paging and caching with Realm
* Setting up Github Actions with secrets and testing

### July
* Releasing posts for Architecture related decisions in Android
* Relearning GraphQL with PokedexGraphQL app

### June
* Architecture related decisions in Android
* Kotlin Coroutines book - Part 4 (Examples, Best practices)

### May
* Caching strategies
* Kotlin Coroutines book - Part 3 (Flows)
* Soft Skills: The Software Developer's Life Manual (Book by John Sonmez)

### April
* Switching back to MVVM for test challenges
* MAterial 3 Compose design system

### March
* HackerRank easy challenges
* Kotlin Coroutines book - Part 2 remaining sections

### February
* Github Pages + Jekyll static website generator

### January
* Using IntelliJ Platform SDK with StringTemplate, JFlex, and Grammar-Kit/EBNF to write a custom language support plugin for templating with StringTemplate over Kotlin

## 2023

### December
* Kotlin Coroutines book - Part 2
* Improved Compose performance

### October, November
* More on Kotlin Multiplatform and SwiftUi basics with Companies House app
* Koin
* Material 3

### September
* Konsist static analysis, plus custom rules for it

### August
* LeakCanary, how to remove all leaks with it

### July
* Decompose
* How to update customised rule sets in Detekt (use git to three-way merge old rules, new rules, and custom rules)

### June
* Scripting basics, dot files, symlinks, aliases, file processing

### March, April, May
* Some more advanced Realm stuff

### February
* Realm basics

### January
* Kotlin generics and variance

## 2022

### December
* Kotlin serialization

### November
* Switch to Version catalogs

### October
* Ktor

### September
* MockK

### August
* Gradle Profiler
  * [https://github.com/gradle/gradle-profiler][gradle-profiler]
* Various caching types
  * [https://proandroiddev.com/caching-in-the-android-build-process-a52641a66b31][caching-types]

### July
* Dagger/Hilt

### June
* PlantUML
* Dependency Analysis plugin

### May
* AndroidX App Startup library

### April
* Bought the book 'Kotlin Coroutines' and went through Part 1
  * [https://leanpub.com/coroutines][coroutines]

### March
* MVIKotlin

### February
* Value classes/inline classes
  * [https://kotlinlang.org/docs/whatsnew15.html#inline-classes][inline-classes]

### January
* Kotlin Native Concurrency

## 2021

### December
* Kotlin Native Concurrency
  * Old memory model (example also with new)
    * [https://play.kotlinlang.org/hands-on/Kotlin%20Native%20Concurrency/00_Introduction][native-concurrency]
  * New memory model migration
    * [https://blog.jetbrains.com/kotlin/2021/08/try-the-new-kotlin-native-memory-manager-development-preview/][new-native-memory-manager]
    * [https://github.com/JetBrains/kotlin/blob/master/kotlin-native/NEW_MM.md][new-native-memory]
  * Native coroutines library
    * [https://github.com/rickclephas/KMP-NativeCoroutines][native-coroutines]

### October, November
* Kotlin Multiplatform with iOS

### August, September
* SwiftUI, iOS basics
* Gradle Kotlin DSL

### July
* Jetpack Compose

### June
* Factory Method pattern (from Design Patterns), static Factory Method pattern (From Effective Java and Effective Kotlin)
  * [https://refactoring.guru/design-patterns/factory-method/java/example][factory-method]
  * [https://blog.kotlin-academy.com/item-30-consider-factory-functions-instead-of-constructors-e1c747fc475][factory-functions]
* Jetpack Compose basics

### May
* Use delegates in Android views to use composition over inheritance and Interface Segregation Principle
* AndroidX WorkManager

### April
* Material theme colour scheme, extra attributes PrimarySurface and OnPrimarySurface, how to extend colour scheme

### March
* Material theme attributes, colour usage

### February
* Manipulate Lottie animations

### January
* Clean Architecture (now also in a Test Challenge: EveryLife Tasks)

## 2020

### December
* Replace ExpandableRecyclerView with Groupie; can remove ViewHolders altogether
  * [https://github.com/lisawray/groupie][groupie]
* Barista
* Switched to JUnit5, used parameterized tests

### November
* Dagger Hilt
* kotlin-result
* FlowBinding instead of RxBinding:
  * [https://dev.to/ychescale9/binding-android-ui-with-kotlin-flow-22ok][flow binding]

### October
* Combine two calls with coroutines asynchronously

### September
* Scabbard for visualizing Dagger dependencies
* Replaced Hamcrest with Kotest

### August
* How to write functional mappers (added to Companies House/FilingHistory)

### July
* Kotlin Multiplatform basics: Ktor, server and web development basics

### June
* SqlDelight advanced usage: many-to-many relations with joint tables
  * [https://github.com/AdevintaSpain/Barista][barista]
* LiveData

### May
* SqlDelight
* Countless Kotlin functions like toCharArray, joinString, isDigit, Double.pow(Double), windowed
* TDD in Unit testing
* Reflection extensions to test private members and functions
* Parameterized Unit tests (AtbashCipher test in Exercism)
* coroutines basics

### April
* Kotlin scan functions
* Kotlin zip functions

### March
* Flipper by Facebook
* Styling with MaterialComponents

### February
* IntelliJ plugin development
* Animations with MotionLayout

### January
* StringTemplate template language
* IntelliJ plugin development basics
* View Binding instead of findViewById or Kotlin synthetics

## 2019

### December
* Process death recovery with MvRx, without savedInstanceState or ViewModel-SavedState
* Kotlin Gradle DSL + how to create and use Gradle Plugins

### November
* How to use subprojects closure to simplify submodule gradle files
* Use private members and functions in Kotlin classes in tests with reflection
* GitLab CI and how it needs source sets added to distinct flavor code for successful builds compared to simple AS builds
* How to create animated vector drawables and how to use them (animated-vector)

### October
* Detekt
* How to use MvRx Async for state management
* Robot pattern for tests

### September
* Navigation component
* How to design module dependencies and dependency graph
* MvRx basics

### August
* More GraphQL (subscriptions)

### July
* GraphQL

### June
* Kotlin idiomatic functions:
  * none(): [https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/none.html][none]
    * For checking if Collection contains no elements with data in the Predicate (contains only looks for objects)
  * any(): [https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/any.html][any]
    * Opposite of the above
* StringRef and ViewModel can be used to add strings to business logic (like in ConnectionsViewModel in MedShr)
* LiveData and MVVM (Connections in MedShr)

### May
* Kotlin idiomatic functions:
  * orEmpty(): [https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/or-empty.html][or-empty]
  * getOrNull(): [https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/get-or-null.html][get-or-null]
  * compareValues: [https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.comparisons/index.html][compare-values]
* Composition and multiple inheritance by Kotlin delegation: 
  * [https://proandroiddev.com/kotlin-pearls-multiple-inheritance-3f4d427141a5][kotlin-delegation]
* Using a shortcut for a ViewModel parameter is problematic when you replace it, like updatedCase in MedShr.
* Recursive generics are not working with Kotlin (MedShr GenericCategory)

### April
* Separation of Network, Data and View model is important
* Use @Intdef and @StringDef instead of enums (MedShr/Polls.kt) Just use enums!!!

### January, February, March
* ViewModel for storing data for Presenter and Activity

## 2018

### October
How to use typealias for deferred actions

[or-empty]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/or-empty.html
[get-or-null]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/get-or-null.html
[compare-values]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.comparisons/index.html
[kotlin-delegation]: https://proandroiddev.com/kotlin-pearls-multiple-inheritance-3f4d427141a5
[any]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/any.html
[none]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/none.html
[groupie]: https://github.com/lisawray/groupie
[flow binding]: https://dev.to/ychescale9/binding-android-ui-with-kotlin-flow-22ok
[barista]: https://github.com/AdevintaSpain/Barista
[factory-method]: https://refactoring.guru/design-patterns/factory-method/java/example
[factory-functions]: https://blog.kotlin-academy.com/item-30-consider-factory-functions-instead-of-constructors-e1c747fc475
[inline-classes]: https://kotlinlang.org/docs/whatsnew15.html#inline-classes
[native-concurrency]: https://play.kotlinlang.org/hands-on/Kotlin%20Native%20Concurrency/00_Introduction
[new-native-memory-manager]: https://blog.jetbrains.com/kotlin/2021/08/try-the-new-kotlin-native-memory-manager-development-preview/
[new-native-memory]: https://github.com/JetBrains/kotlin/blob/master/kotlin-native/NEW_MM.md
[native-coroutines]: https://github.com/rickclephas/KMP-NativeCoroutines
[coroutines]: https://leanpub.com/coroutines
[gradle-profiler]: https://github.com/gradle/gradle-profiler
[caching-types]: https://proandroiddev.com/caching-in-the-android-build-process-a52641a66b31
