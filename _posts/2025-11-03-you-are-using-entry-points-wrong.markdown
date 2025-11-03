---
layout: post
title:  "You are using Entry Points wrong"
date:   2025-11-03 19:50:00 +0000
categories: android dagger hilt dependency_injection
mermaid: false
---

![starting-image](/assets/img/posts/20251103_you_use_entry_points_wrong.jpg){: width="1280" height="896" }

This is my first article where I simply regurgitate the documentation with some additional information. But I do it for a good reason. It seems everyone – including me – got this wrong. Maybe it’s because I haven’t worked with the right people or the right types of projects, and that might be true. But I haven't come across many examples of code using it correctly.

First, I’ll explain when and why you should use Entry Points. Then, in the second part, I’ll talk about the best way to use them and where people often go wrong. Don’t worry, I’ve made those mistakes myself too, which is why I decided to write this article.

I will assume that you already know the basics of Dagger/Hilt and [Entry Points][entry-points].

## When to use Entry Points?

This is the easier part, and most people get this right, even though it's often overused. But for completeness I will go through this as well.

### Where not to use them because you are already covered

#### @AndroidEntryPoint

As documented [here][androidentrypoint], **@AndroidEntryPoint uses @EntryPoint** behind the scenes, so you don't need the boilerplate for the following classes:

* Activity
* Fragment
* View
* Service
* BroadcastReceiver

This is provided by the **hilt-android** library, which is **closely** related to **Dagger/Hilt**.

#### HiltViewModel, Navigation and HiltWorker

As documented [here][viewmodel], ViewModels should be constructor injected by using the HiltViewModel annotation. This will allow you to retrieve them by using the ViewModelProvider API.

What's more, you can inject NavGraph-scoped ViewModels and Workers with Hilt, as described [here][worker]. These are provided by **AndroidX libraries**, which are also made by Google, but they **aren't directly connected**. It's interesting to see how they set the boundaries, maybe because @AndroidEntryPoint uses @EntryPoint directly, whereas the AndroidX classes only work with hilt-android. I haven't looked into this any further.

### Where not to use them for performance reasons

They have a small overhead which quickly grows not so small if you pay it many times -- in list items. So **don't use them in ViewHolder items or classes that are used in list items**. At least don't if the list has an indeterminate number of items.

### Where to use them

**Everywhere else**, of course. But in a well-written code base you have to use them very rarely. So always **ask the question if you really need them** or not.

So far I have found two valid use cases for Entry Points.

The **first** one is **when you refactor a large legacy code base**, specifically when you want to convert manual or no dependency injection to Dagger and Hilt. You have most of your dependencies injectable, but a lot of dependents are not ready to be refactored yet. For example, you have large manual DI modules, presenters, and so on. To make use of your already available dependencies, you convert the manual usages to them.


```kotlin
object ManualFooModule {
  
  fun foo() : Foo {
    return Foo(
      SomeOtherModule.bar(),
      SomeOtherModule.baz(),
    )
  }
}

```
{: file='ManualFooModule.kt before'}

You need to provide Foo for some dependents, because they cannot be injected yet, but Bar and Baz are already injectable. In this case you can have an intermediate state, where the manual injection uses dependencies on the graph: 

```kotlin
object ManualFooModule {
  
  fun foo() : Foo {
    return Foo(
      someEntryPoint().bar(),
      someEntryPoint().baz(),
    )
  }
}

```
{: file='ManualFooModule.kt after'}

And then when nobody uses ManualFooModule.foo() anymore, you can simply delete it.

Don't forget that this is mostly **temporary**. Once your class is completely converted, you don't need the Entry Points anymore, so delete them too.

The **second usage is third-party code with manual initialisation**. But not all third-party code is created equal. Most of the time you can add a **wrapper interface** and inject the wrapper. No need for Entry Points. Using the interface has the additional benefit of the ability to mock them in tests. 'Additional' from our point of view, of course. This would be the main benefit if we wouldn't talk about Entry Points.

This is how you can mostly **avoid Entry Points** in this case:

```kotlin
interface ThirdPartyWrapper {
  fun doSomething()
}

class ThirdPartyWrapperImpl(thirdParty: ThirdParty) {
  override fun doSomething() {
    thirdParty.doSomething()
  }
}

@Module
@InstallIn(SingletonComponent::class)
class ThirdPartyWrapperModule {

  @DefaultDispatcher
  @Provides
  fun providesThirdPartyWrapper(@ApplicationContext context: Context): ThirdPartyWrapper {
    return ThirdPartyWrapperImpl(ThirdParty.init(context))
  }
}

```
{: file='ThirdPartyWrapper.kt'}

Now let's see an example when we do need an Entry Point in the above-mentioned third-party code case.

**ContentProvider** is a class where Google doesn't provide an injectable solution yet.

Similarly to an Activity we cannot use constructor injection. In the absence of an out-of-the-box solution, we cannot use field injection either. So the only viable option is to use Entry Points.

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
internal interface MyContentProviderEntryPoint {
  fun moviesRepository(): MoviesRepository
}
```
{: file='MyContentProviderEntryPoint.kt'}

```kotlin
class MyContentProvider : ContentProvider() {

  // @Inject doesn't work here
  private lateinit var moviesRepository: MoviesRepository

  override fun onCreate(): Boolean {
    val entryPoint = EntryPointAccessors.fromApplication(
      context!!,
      MyContentProviderEntryPoint::class.java
    )

    // Manually "inject" the dependency by calling the method on the EntryPoint
    this.moviesRepository = entryPoint.moviesRepository()

    return true
  }

	// ... other required methods
}

```
{: file='MyContentProvider.kt'}

Another good example is RecyclerView Adapters. Above I said that you shouldn't use Entry Points in list items, like ViewHolders, but you should use them in Adapters. This is how it would look if I would still use RecyclerView in [FlickSlate][FlickSlate]:

```kotlin
@EntryPoint
@InstallIn(ActivityComponent::class)
interface ViewHolderEntryPoint {
  fun imageLoader(): ImageLoader
}
```
{: file='ViewHolderEntryPoint.kt'}

```kotlin
class MovieViewHolder(
  view: View,
  private val imageLoader: ImageLoader // Dependency passed in constructor
) : RecyclerView.ViewHolder(view) {

  private val posterImageView: ImageView = itemView.findViewById(R.id.posterImageView)

  fun bind(movie: Movie) {
    imageLoader.load(view.context, movie.posterUrl, posterImageView)
  }
}
```
{: file='MovieViewHolder.kt'}
```kotlin
class MoviesAdapter(
  private val context: Context // Pass context to the adapter
) : RecyclerView.Adapter<MovieViewHolder>() {

  private val entryPoint: ViewHolderEntryPoint by lazy {
    EntryPointAccessors.fromActivity(
      context, // The context must be an Activity
      ViewHolderEntryPoint::class.java
    )
  }

  override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MovieViewHolder {
    val view = LayoutInflater.from(parent.context)
      .inflate(R.layout.item_movie, parent, false)

    val imageLoader = entryPoint.imageLoader()

    return MovieViewHolder(view, imageLoader)
  }

  // ... other adapter methods
}
```
{: file='MoviesAdapter.kt'}

I had to give this example instead of a Compose one, because Compose doesn't need this. You create your dependencies in your ViewModel and pass them to your Composables, for which **you are always in control**.

## Where to add the Entry Points?

Now onto the main part which everyone I know got wrong. That's not a grave error anyway, but it definitely **makes refactoring harder**, which is the likely reason you need them at all.

#### Bad practice

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object FooModule {
  @Provides
  fun provideFoo(): Foo {
    return Foo()
  }

  @EntryPoint
  @InstallIn(SingletonComponent::class)
  interface FooInterface {
    fun foo(): Foo
  }
}
```
{: file='Bad practice'}

The Entry Point is in a **separate file and close to its target**. Your instinct says this is similar to a **Provider** which should in fact be near the target implementation in a separate file. But the matter of fact is that **this is not a Provider, but uses providers, so it's better used slightly differently**.

#### Good practice

```kotlin
class MyClass : NonHiltLibraryClass() {

  @EntryPoint
  @InstallIn(SingletonComponent::class)
  interface MyClassInterface {
    fun foo(): Foo

    fun bar(): Bar
  }

  fun doSomething(context: Context) {
    val myClassInterface =
        EntryPoints.get(applicationContext, MyClassInterface::class.java)
    val foo = myClassInterface.foo()
    val bar = myClassInterface.bar()
  }
}
```
{: file='Best practice'}

Here the Entry Point is so **close to the usage** that it is actually implemented and instantiated right in the class. Why is this favourable? Instantiating Entry Points is **not the most expensive** construct, especially if you use it in moderation as you should. Compiling and instantiating the dependencies is much more expensive.

Keeping them nearby to where they are used **gives you more control over when to create them, what to include, and when to remove them**. In my experience, using poor practices leads to lots of unused items in the Entry Points, or even entire Entry Points that aren’t needed. But when you place them in the classes that depend on them, and then you refactor them, it is just natural to delete the Entry Point as well.

Finally, when you work on modularisation, **they might become a blocker**, or at least a source of additional work. If you can just move them with your class, your job just became much easier. Mostly you will remove them before you move anything, but it's not always possible.

You can also read about bad and good practices in the **official documentation** [here][entry-points].

## Conclusion

As you have learnt today, the rules about Entry Points are simple, but you have to use your judgement. Use them in third-party classes, which you cannot inject or wrap. They are very useful in large legacy code bases, which you cannot refactor easily. Otherwise try to avoid them whenever possible.

[androidentrypoint]: https://dagger.dev/hilt/android-entry-point.html
[viewmodel]: https://dagger.dev/hilt/view-model.html
[worker]: https://developer.android.com/training/dependency-injection/hilt-jetpack
[FlickSlate]: https://github.com/herrbert74/FlickSlate
[entry-points]: https://dagger.dev/hilt/entry-points.html