---
layout: post
published: true
title: AppCompat 1.2 Lint Checks - AppCompatResources or ContextCompat or ResourcesCompat
date: '2020-10-20'
share-img: "assets/img/jetpack-hero.svg"
tags: [android, appcompat, androidx, jetpack]
---
[AppCompat 1.2](https://developer.android.com/jetpack/androidx/releases/appcompat#1.2.0-beta01) release came with a couple of new lint rules which suggest using either `AppCompatResources` or `ContextCompat` or `ResourcesCompat` depending on the API you were originally consuming. But what is the difference between the common methods in these three classes? 


## What are Compat Classes?

The AndroidX/Jetpack libraries provide a bunch of `compat` classes to help developers work with deprecated API’s and avoid having to write `if (Build.VERSION.SDK_INT >= X)` everywhere. A few `compat` classes even backport newer platform capabilities to older platforms (more on that later). 

A general rule of thumb that has served me extremely well over the last few years is to look for a `Compat` class whenever I encounter a deprecated API method or try to use a method that is only available in the recent versions of the platform framework API. 


![Min-API](/assets/img/min_api.png)


![Deprecated](/assets/img/deprecated.png)


The name of the `compat` class is usually the same as the class in which the original API method exists. For the above example, `setTextAppearance` is a part of the `TextView` class. Therefore, the equivalent `compat` class which **might** (because not all such methods have a `compat` equivalent) have this method is the [TextViewCompat](https://developer.android.com/reference/androidx/core/widget/TextViewCompat#setTextAppearance(android.widget.TextView,%20int)) class. Note that it is important to look at the class which contains the API method and not mistake it for the class on which we are operating. For example, the `setBackgroundDrawable` method is deprecated and we could be using it to set a background on a `Button`, but the method actually belongs to the [View](https://developer.android.com/reference/android/view/View#setBackgroundDrawable(android.graphics.drawable.Drawable)) class and hence it’s equivalent would be found in the [ViewCompat](https://developer.android.com/reference/androidx/core/view/ViewCompat#setBackground(android.view.View,%20android.graphics.drawable.Drawable))` class.

Now that we have a primer on how to find `compat` equivalents, let's take a look at the methods inside `ContextCompat` and `ResourcesCompat`. For this post, we’ll only look at the methods which have the same name in both classes.


![ContextCompat-vs-ResourcesCompat](/assets/img/contextcompatVSresourcescompat.png)


From the above table, we can see that there is no real difference between the `ContextCompat` methods vs the ones in `ResourceCompat`. You would use `ContextCompat` methods if you had access to `Context` but if you only had access to the `Resource` object, then you would use the ones inside `ResourceCompat`. At the end of the day, the result from both of them will be the same.

At this juncture, it is important to remember that platform framework support for Vector Drawables and tinting was added in API level 21. As the `ContextCompat` and `ResourcesCompat` methods simply delegate to platform framework, these methods **will not** be able to load Vector Drawables on older platforms as the older platforms don’t have support for them.


## Enter AppCompat

Now the examples that were detailed above mostly save developers from having to write `if (Build.VERSION.SDK_INT >= X) `but `appcompat` is different. `AppCompat` actually works on backporting the new UI toolkit functionality that was introduced in the later platform versions, such as Vector Drawables, tint, theme attributes in ColorStateLists, etc. So if we want to load VectorDrawables on older platforms, we would need to use the AppCompat library. The hook into the AppCompat library (through Java/Kotlin code) is through [AppCompatResources](https://developer.android.com/reference/androidx/appcompat/content/res/AppCompatResources). Now’s let’s compare the methods inside `AppCompatResources` with those inside `ContextCompat` and `ResourcesCompat`

**`static Drawable getDrawable(...)`:**


<table>
  <tr>
   <td>
   </td>
   <td>ContextCompat
   </td>
   <td>ResourcesCompat
   </td>
   <td>AppCompatResources
   </td>
  </tr>
  <tr>
   <td>Vector Drawable (API Level 21 & above)
   </td>
   <td>Yes
   </td>
   <td>Yes
   </td>
   <td>Yes
   </td>
  </tr>
  <tr>
   <td>Vector Drawable (API Level 20 & below)
   </td>
   <td>Crash
   </td>
   <td>Crash
   </td>
   <td>Yes
   </td>
  </tr>
</table>

**`static ColorStateList getColorStateList(...)`:**


<table>
  <tr>
   <td>
   </td>
   <td>ContextCompat
   </td>
   <td>ResourcesCompat
   </td>
   <td>AppCompatResources
   </td>
  </tr>
  <tr>
   <td>Theme attributes in CSL (API Level 21 & above)
   </td>
   <td>Yes
   </td>
   <td>Yes
   </td>
   <td>Yes
   </td>
  </tr>
  <tr>
   <td>Theme attributes in CSL (API Level 20 & below)
   </td>
   <td>Crash
   </td>
   <td>Crash
   </td>
   <td>Yes
   </td>
  </tr>
</table>

&nbsp;

## FAQ’s

*   **Why does the lint check recommend using `ContextCompat` and not `AppCompatResources` if `ContextCompat` can crash on Pre API Level 21 devices?**

As of AppCompat 1.2, the lint check recommends using `ContextCompat`. This is a [known bug](https://issuetracker.google.com/issues/165927862) and will be resolved soon. Future versions of AppCompat will instead recommend using `AppCompatResources` to load drawables and Color State Lists.

*   **Should we use `AppCompatResource` if we only support API Level 21 and above?**

Yes, we should. `VectorDrawableCompat` is an unbundled implementation of Vector Drawables outside of the framework and can be updated without needing platform framework updates. This allows developers to work around any bugs that are there in the framework implementations of Vector Drawables (which it did have).

*   **`ViewCompat` has a `setBackgroundTintList` method which applies tints to views even on pre API Level 21 devices. How does that work?**

The ability to tint backgrounds and images was added in API Level 21. As explained earlier in the post, the ViewCompat class does not backport this functionality to older API platforms. That is still the job of `AppCompat`. `ViewCompat` uses a nifty trick where on the older platform (pre-API level 21), [it checks if the view implements `TintableBackgroundView`](https://github.com/androidx/androidx/blob/androidx-master-dev/core/core/src/main/java/androidx/core/view/ViewCompat.java#L2787). If it does, then the `ViewCompat` delegates to it to perform the tinting. Most of the [widgets](https://github.com/androidx/androidx/blob/androidx-master-dev/appcompat/appcompat/src/main/java/androidx/appcompat/widget/AppCompatButton.java#L59) in `AppCompat` implements the `TintableBackgroundView` interface and offer tinting functionality, effectively backporting it to older platforms.

Since we use the AppCompat theme (or MDC theme) in our apps, we end up using the AppCompat widgets and hence get the tinting capability through `ViewCompat`.

>Note: The same mechanism applies for `ImageCompat`
{: .prompt-info }

*   **What about `app:srcCompat`, `app:drawableLeftCompat`, `app:tint`, etc.?**

Drawables passed through the above XML attributes are inflated by `AppCompatResource`. Tint passed through the above XML attributes are also applied by the app compat widgets.

>Note: Just like the framework had bugs with Vector Drawables, the framework, unfortunately, had bugs with tinting as well. Hence it is always recommended to use `app:srcCompat`, `app:tint`, etc. even when we are only targeting API Level 21 and above.
{: .prompt-info }

*   **What is the future with Jetpack Compose?**

Jetpack Compose is a full-fledged replacement for the UI toolkit. Once it’s stable, I believe we would no longer need the AppCompat Widgets, AppCompatResources, ContextCompat and ResourcesCompat. Till then (and even in the future when maintaining existing apps), it would serve us well to know when to use which method and how things work under the hood.