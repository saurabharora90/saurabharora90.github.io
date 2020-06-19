---
layout: post
published: true
title: Modular architecture- Using Dagger Multibinding to initialize modules
date: '2020-06-19'
tags: [android, modular, dagger]
---
As Android codebases continue to become increasingly modular, some of these modules might need to be initialized at app startup in order to wire everything up. This usually results in the `Application` class or the `Splash/Main` activity, calling the “managers/helpers” of these modules, providing them with the dependencies, and asking them to initialize themselves. If an app has a few modules, then this might not be an issue but as the number of modules increase, this management of their initialization can quickly become unwieldy. 

Through this post, we’ll explore how we can use Dagger multibindings to make this easier. If you aren’t familiar with Multibindings and `@IntoSet/@IntoMap`, then I would advise reading the [documentation](https://dagger.dev/dev-guide/multibindings.html) before proceeding. I’ll wait.

*Disclaimer*: This is only for modules that are self-contained and need to be informed about app launch.


### Create a `ModuleInitializer` interface

First up, we’ll create a `ModuleInitializer` interface that will be implemented by any gradle module that needs to be instantiated at app startup. We’ll add this interface to our “shared” module (a common module on which every module depends)

<script src="https://gist.github.com/saurabharora90/b3888a261e0a3000d1ea589bac0ee230.js"></script>

The interface only exposes one method, `initialize` which returns a `Completable`. If the module creation is synchronous or it doesn’t need to block the app launch process, one can simply return `Completable.complete`. Using a Completable allows us to optionally wait for all the module initialization to complete before proceeding with the app launch process.

Next, any gradle module that needs an initialization step, can implement this interface and define its dependencies that will be provided by Dagger. For example, at [Viki](https://www.viki.com/), we have a `customercare` module that houses all our fragments and activities for customer support and uses the Zendesk SDK internally. The Zendesk SDK needs to be initialized at app launch. Similarly, we also have a `subscription` module that needs to establish a connection with Google Play Store at launch.

<script src="https://gist.github.com/saurabharora90/846ac70042eb143d9ea772f0adaa24c4.js"></script>



### Create a Set of Initializers

Now in the app module, we will create a new dagger module that will provide these ModuleInitializers. Through multibinding, Dagger assembles the collection so that the application code can inject it without depending directly on the individual bindings. We’ll also update our AppComponent to provide this set of `ModuleInitializer`

<script src="https://gist.github.com/saurabharora90/e6951bd8296a99889557e6866af6c58b.js"></script>

Now in our Application or Splash/Main Activity, we can get this set of `ModuleIntializers` and initialize each of them. If all modules can be initialized independently, you can use `Completable.merge` to kick off their initialization process.

<script src="https://gist.github.com/saurabharora90/d256c257454f1f7b2f417a466360c661.js"></script>


If your modules need to be initialized sequentially, then we can use `concatMap` instead of merge.

### What if `customercare` depends on `subscription` to be ready?

To support a gradle module being dependent on the prior initialization of another gradle module, we’ll update the `ModuleInitializer` and add a `dependencies` method to the interface (which by default returns an empty list, i.e. no dependencies)

<script src="https://gist.github.com/saurabharora90/30cf9eda9b7d05519ca2ca411ed96bbe.js"></script>

Next, we’ll update the `CustomerCareInitializer` to return `SubscriptionInitializer` as its dependency.

<script src="https://gist.github.com/saurabharora90/709f5d68e3d221db327253220dd3a801.js"></script>

Now instead of initializing every module from the set, we need to check the dependencies of each module, and if the dependency hasn’t been initialized yet, we need to initialize it before we initialize the module itself. Since the dependencies is a list of `class`, we would need to update our Dagger Module to Bind into a `map` rather than into a `set`. Here the map key will be the `class` of the corresponding `ModuleInitializer`

<script src="https://gist.github.com/saurabharora90/ffbcd1d04d5f4bff41c76296fd1c7cc0.js"></script>

Lastly, we’ll need to update how we generate the initializing Completable that is executed at app launch. Instead of iterating over the set, we now have a map and each module has its own list of dependencies. Therefore, we will iterate through each entry in the map, look at the dependencies, and if there are dependencies, we will first initialize them. For every module we initialize, we will also update a local store of initialized modules. Every time we need to initialize a module, we’ll first check the local store to make sure this module hasn’t already been initialized as a part of another dependency. This will prevent us from initializing the same module twice (in case multiple modules initializers depend on a single module initializer).

<script src="https://gist.github.com/saurabharora90/2795b0ff9000e333f1db7fdcd165a12b.js"></script>

In the future, if we want to add a new module that requires initialization at app launch, we’ll create a new implementation of the `ModuleInitializer` interface inside the new module and bind that to the map/set in the dagger `InitializerModule`. The place where we iterate over the map/set and instantiate each module will not need any changes, making this process a lot more manageable.


### How does this compare to JetPack App Startup

[App Startup](https://developer.android.com/topic/libraries/app-startup) is a new Jetpack library that was recently announced by Google. From the docs:

>The App Startup library provides a straightforward, performant way to initialize components at application startup. Both library developers and app developers can use App Startup to streamline startup sequences and explicitly set the order of initialization.

While this is a great initiative and hopefully will be adopted by libraries such as Firebase, Facebook, and others (which use content providers to instantiate themselves), the current API surface is not sufficient for app developers who want to instantiate their own modules. The library uses a content provider to provide a way to call each of our component initializers. Content providers are automatically created by the system and applications usually do not create `ContentProvider` instances directly. As a result, Content Provider will be created before the call to [Application#onCreate](https://developer.android.com/reference/android/app/Application.html#onCreate()) as mentioned in the docs: 

>Called when the application is starting, before any activity, service, or receiver objects (excluding content providers) have been created.

Therefore, it is not possible to provide these component initializers, defined by the App Startup Library, with dependencies from your dagger graph as most applications create their AppComponent in `Application#onCreate`.


## Conclusion

As we make our codebases modular, we don’t want to bloat up our Application class with logic on instantiating these modules. Changes to the module internals might cascade to the app module as it might need different dependencies, or the module initialization might change from being synchronous to asynchronous or vice-versa.
Dagger Mulibinding can provide a good alternative to manually instantiating individual modules at app launch.

Big thanks to [Márton Braun](https://twitter.com/zsmb13) for reviewing the post.

That's it! If you have any questions or suggestions, leave a comment below. If you want to be notifed of such posts, please follow me on [Twitter](https://twitter.com/saurabh_arora90).



