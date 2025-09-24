---
layout: post
title:  "Make everything internal"
date:   2025-09-24 19:50:00 +0000
categories: android architecture modularisation
mermaid: true
---

![starting-image](/assets/img/posts/20250924_make-everything-internal.jpg){: width="1024" height="701" }

This article is about how we can make everything internal in a project supported by Hilt and Dagger. At least in the data layer. And in smaller applications, like my [FlickSlate][FlickSlate] repository.

You see, I always dismissed -- at least for the most part -- making interfaces and classes internal, because they needed to be open for dependency injection and testing. Why bother then?

But recently my colleagues introduced a lint warning about internal classes. At first I was in denial, especially because no one asked me about this. And then I asked myself, why not? What's exactly stopping us? What are the limits? I wanted answers, so I started converting everything internal or private.

### Where to start?

I probably shouldn't start in the [shared or base][Super Layers] layer, since not much should be kept hidden there. The same goes for the domain layer. That leaves us with the data layer and the UI layer. I picked the data layer because it's easy to see what details should be kept inside the modules and what should be accessible from outside.

As I explained in [my article about the Data layer][Data layer repository], the repository keeps the details of how data is handled separate from the main part of the app or the user interface, especially when it's used directly in the ViewModels. So, I decided to keep everything inside the repository private, except for the parts I call Accessors. That's how I refer to the different implementations of the repository.

### Obstacles

I can think of two common issues, and I actually faced both of them while doing this task. Both problems are related to dependency injection. The first one is about how to allow a public Hilt module to reveal an internal part of the code.

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
interface TvRepositoryModule {

  @Binds
  //error: 'public' function exposes its 'internal' parameter type 'TvAccessor'.
  fun bindTvRepository(impl: TvAccessor): TvRepository
}
```
{: file='failing TvRepositoryModule.kt'}

Because the module needs to be public, right? That's true, but with a twist. Dagger modules need to be public to be able to use subcomponents and assemble them in the Application. But the best practice for Hilt modules is actually to restrict visibility; at least the bottom of [this page][Hilt modules] says so.

But it doesn't mention anything about test overrides, which is the second problem. How to access and use these internal dependencies and their modules in tests? We have to make them public, which means the second makes the first issue valid.

Google has a few guidelines about Dagger and Hilt, like [this][Google on Hilt modules], but it's useless as usual for non-trivial problems like ours. By all means, I don't think that the problems above are highly exotic, but I have never seen a solution or anyone talking about this anywhere.

So let's solve this.

### Solution for the first problem

For the first problem [this article][hide internal from public dagger] seemed to help. It is about Dagger, but the principle is the same.

It uses the fact that while a module mustn't expose an internal class, it doesn't expose an internal module. So you can wrap the internal module in a public one.

> ##### A note on Auto Dagger
>
> In my case this meant that I had to remove some @AutoBind annotations. You probably use the @TestInstallIn annotation from Hilt. I automated this with [Auto Dagger][Auto Dagger], which uses @AutoBind and @Replaces to generate the bindings. Obviously, for this solution to work, I need to declare the module manually so I can override them.
{: .block-tip }

After applying the above changes my modules looked like this:

```kotlin
@Module(includes = [InternalTvRepositoryModule::class])
@InstallIn(ViewModelComponent::class)
interface TvRepositoryModule

@Module
@InstallIn(ViewModelComponent::class)
internal interface InternalTvRepositoryModule {

  @Binds
  fun bindTvRepository(impl: TvAccessor): TvRepository

}
```
{: file='failing TvRepositoryModule.kt during UI tests'}

This went surprisingly well for the app build. It built and ran just fine. But the second problem started to emerge when I tried to run the UI tests, which require fake repositories.

The problem is that @TestInstallIn will override the public module but not the internal, so it results in duplicate bindings because the internal one still counts as a binding.

### First attempt on the second problem with testFixtures

To alleviate the above issue, I tried to move the test doubles to testFixtures. My idea was that from testFixtures I can still override the internal implementations and provide them for UI tests.

Unfortunately this didn't work out due to [this issue][ksp-test-fixtures] in ksp. It is simply not working at all for testFixtures. Since it hadn't been dealt with for a year, I had to find another solution.

### Named bindings to the rescue

The solution was found in [Stack Overflow][internal solution]. The trick is to make the internal binding a named binding and then rebind it WITHOUT the name. This way the outer binding uses the original, but because of the qualifier, it's distinct from the original, so this means no error message about double binding.

```kotlin
@Module(includes = [InternalTvRepositoryModule::class])
@InstallIn(ViewModelComponent::class)
interface TvRepositoryModule {

  @Binds
  fun bindTvRepository(@Named("Internal") impl: TvRepository): TvRepository

}

@Module
@InstallIn(ViewModelComponent::class)
internal interface InternalTvRepositoryModule {

  @Binds
  @Named("Internal")
  fun bindTvRepository(impl: TvAccessor): TvRepository

}

```
{: file='passing TvRepositoryModule.kt'}

And now for tests you can override like this:

```kotlin
// This needed to be deleted
// @Replaces(TvAccessor::class)
@ViewModelScoped
class FakeTvRepository @Inject constructor() : TvRepository {

  ...

}
```
{: file='FakeTvRepository.kt'}

```kotlin
@Module
@TestInstallIn(
  replaces = [TvRepositoryModule::class],
  components = [ViewModelComponent::class]
)
interface FakeTvRepositoryModule {

  @Binds
  fun bindTvRepository(impl: FakeTvRepository): TvRepository

}
```
{: file='FakeTvRepositoryModule.kt'}

### Profiling the difference

I haven't used profiling in a while, but I wanted to refresh my knowledge on it, and I was curious about the benefits of internalising everything, so I decided to try and compare the results with [gradle profiler][gradle profiler].

Unfortunately it's currently clashing with the dependency analysis plugin, [version 0.22.0 doesn't work with Gradle 9.0][gp kotlin 9.0], and to set gradle properties, [you need to add them to gradle.properties as well (command line param not enough!!!), which is undocumented][gp params]. But after a few hours of tinkering, I was able to run the profiler.

The result was a bit underwhelming but could have been expected in retrospect. All the scenarios brought around 1-4% improvements, from deep ABI changes through non-ABI changes to clean builds.

# Conclusion

But every little helps! And it was fun to figure out all the problems and add profiling to this project as well.

[BaBeStudios-Base]: https://bitbucket.org/babestudios/babestudiosbase/src/master/
[BaBeStudios-Base-Maven]: https://mvnrepository.com/artifact/io.bitbucket.babestudios
[FlickSlate]: https://github.com/herrbert74/FlickSlate
[Super Layers]: https://herrbert74.github.io/posts/architectural-evolution-of-an-app/#super-layers
[Data layer repository]: https://herrbert74.github.io/posts/architecture-data-layer/#repository-layer-optional
[Auto Dagger]: https://auto-dagger.ansman.se/latest/
[Hilt modules]: https://dagger.dev/hilt/modules
[Google on Hilt modules]: https://developer.android.com/training/dependency-injection/hilt-multi-module
[hide internal from public dagger]: https://medium.com/@blackgin/a-simple-trick-to-hide-internal-code-from-a-public-dagger-module-91bce95be463
[internal solution]: https://stackoverflow.com/questions/70521749/hilt-testing-replace-internal-hilt-module-in-separate-android-multi-module-app
[ksp-test-fixtures]: https://github.com/google/ksp/issues/2093
[gradle profiler]: https://github.com/gradle/gradle-profiler
[gp kotlin 9.0]: https://github.com/gradle/gradle-profiler/issues/568
[gp params]: https://github.com/gradle/gradle-profiler/issues/446