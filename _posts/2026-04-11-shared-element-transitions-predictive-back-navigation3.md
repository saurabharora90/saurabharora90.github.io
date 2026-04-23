---
layout: post
published: false
title: "Shared Element Transitions with Predictive Back"
date: '2026-04-11'
categories: [ Compose, Navigation ]
tags: [ android, compose, kmp, navigation3 ]
image:
  path: /assets/img/banner_compose.png
  alt: Jetpack Compose
---

Shared element transitions are a seamless way to transition between composables that have content that is consistent between them. They are often used for navigation, allowing you to visually connect different screens as a user navigates between them.

Recently, while working on them with Navigation 3, I ran into an issue where the default predictive back behavior creates a visual conflict where the screen's shrink animation fights with the shared element's morph animation.

This post walks through the full setup for cross-screen shared element transitions in Nav3, explains why the default predictive back transition clashes with them, and shows the fix.

## Building the Shared Element Transition

Let's start with the scenario. We have two screens:

1. A **list screen** with image tiles
2. A **fullscreen viewer** that displays the tapped image

When the user taps a tile, we want the image to animate from its position in the list to fill the entire screen. When they go back, the image should animate back to the tile. Both screens need to share the same `SharedTransitionScope` and use a matching key on the `sharedElement` modifier.

### Providing the SharedTransitionScope

In Nav3, the `SharedTransitionLayout` lives at the root of your app (typically in your Activity). Both screens need access to the `SharedTransitionScope` it provides. In a multi-module project, the screens live in separate feature modules and can't directly reference the Activity. Instead of prop-drilling the scope through every composable, we provide it via a `CompositionLocal`:

```kotlin
val LocalSharedTransitionScope = compositionLocalOf<SharedTransitionScope?> { null }
```

Then, at the root:

```kotlin
SharedTransitionLayout {
    CompositionLocalProvider(LocalSharedTransitionScope provides this@SharedTransitionLayout) {
        NavDisplay(
            backStack = navigator.backStack,
            onBack = { navigator.goBack() },
            sharedTransitionScope = this@SharedTransitionLayout,
            entryProvider = entryProvider { /* routes */ },
        )
    }
}
```

### Wiring Up the Screens

On the list tile, we apply the `sharedElement` modifier to the image. The key (`"media-${id}"`) must match between the source and destination:

```kotlin
@Composable
fun ListTile(post: Post, onClick: () -> Unit) {
    val sharedTransitionScope = LocalSharedTransitionScope.current
    val animatedContentScope = LocalNavAnimatedContentScope.current

    val sharedModifier = if (sharedTransitionScope != null) {
        with(sharedTransitionScope) {
            Modifier.sharedElement(
                rememberSharedContentState("media-${post.mediaId}"),
                animatedVisibilityScope = animatedContentScope,
            )
        }
    } else {
        Modifier
    }

    AsyncImage(
        model = post.mediaUrl,
        modifier = Modifier.size(160.dp).then(sharedModifier).clickable { onClick() },
        contentScale = ContentScale.Crop,
    )
}
```

> `LocalNavAnimatedContentScope` is the Nav3-provided composition local that gives you the `AnimatedVisibilityScope` needed for shared element coordination. It's the Nav3 equivalent of the `this@composable` scope from Nav2's `NavHost`.
{: .prompt-info }

The fullscreen viewer applies the same modifier with the same key:

```kotlin
@Composable
fun FullscreenViewer(post: Post) {
    val sharedTransitionScope = LocalSharedTransitionScope.current
    val animatedContentScope = LocalNavAnimatedContentScope.current

    val sharedModifier = if (sharedTransitionScope != null) {
        with(sharedTransitionScope) {
            Modifier.sharedElement(
                rememberSharedContentState("media-${post.mediaId}"),
                animatedVisibilityScope = animatedContentScope,
            )
        }
    } else {
        Modifier
    }

    AsyncImage(
        model = post.mediaUrl,
        modifier = Modifier.fillMaxSize().then(sharedModifier),
        contentScale = ContentScale.Crop,
    )
}
```

With this in place, tapping a tile navigates to the viewer with a smooth shared element transition. The image morphs from its tile size to fullscreen. Pressing a close button (which calls `navigator.goBack()`) reverses the animation.

![close_press.gif](/assets/img/close_press.gif){: w="320" h="4" }

## The Problem

This setup works perfectly for standard navigation. However, the predictive back gesture introduces a visual clash.

To understand why, it helps to know how shared elements are rendered during a transition. When a match is found (two composables with the same key on different screens), the framework removes the element from both screens and renders a single copy in a **transparent overlay** that sits above the `AnimatedContent`. This overlay copy animates its bounds from the source position to the target position (or vice versa on back). Where the element normally sits on each screen, an invisible **placeholder** reserves the space so surrounding layout doesn't collapse.

Everything that is _not_ a shared element stays inside the `AnimatedContent` and is subject to the **`ContentTransform`**. This is the animation that `NavDisplay` applies to each screen during a transition. For predictive back, the default `ContentTransform` is a seekable scale-and-translate that progressively shrinks the exiting screen as you swipe from the edge.

![broken_predictive_back.gif](/assets/img/broken_predictive_back.gif){: w="320" h="4" }

So during predictive back, the user sees both layers at once: the shared element morphing cleanly in the overlay, and the rest of the exiting screen (background, buttons, progress bars) visibly shrinking underneath it via the `ContentTransform`. The shared element is doing exactly what it should, but the parent's scale transform running alongside it makes the result look noisy and unpolished.

> Programmatic back (`navigator.goBack()`) doesn't have this issue because it triggers a standard pop transition without any scale or translation transform on the parent.
{: .prompt-info }

## The Solution

Since the visual conflict comes from the default `ContentTransform` scaling the non-shared content, the fix is to replace it with something that doesn't compete with the morph. We override `NavDisplay.PredictivePopTransitionKey` in the route metadata, swapping the scale for a simple fade:

```kotlin
entry<FullscreenViewerRoute>(
    metadata = metadata {
        put(NavDisplay.PredictivePopTransitionKey) {
            ContentTransform(
                targetContentEnter = fadeIn(),
                initialContentExit = fadeOut(),
            )
        }
    }
) { route ->
    FullscreenViewer(post = route.post)
}
```

With a fade, the non-shared content simply becomes transparent as the gesture progresses. No scale, no translation. The shared element in the overlay is the only visible motion, producing a clean result.

![fixed_predictive_back.gif](/assets/img/fixed_predictive_back.gif){: w="320" h="4" }

Apply this metadata to every route that participates in shared element transitions.

## Debugging with LookaheadAnimationVisualDebugging

If you're troubleshooting shared element issues, Compose 1.11 introduced `LookaheadAnimationVisualDebugging`. Wrap your `SharedTransitionLayout` with it to see target bounds, animation trajectories, key labels, and match counts drawn directly on the UI:

```kotlin
@OptIn(ExperimentalLookaheadAnimationVisualDebugApi::class)
LookaheadAnimationVisualDebugging(isShowKeyLabelEnabled = true) {
    SharedTransitionLayout {
        // ...
    }
}
```

This is what helped confirm that the shared element was seeking correctly on its own, and the visual conflict was caused by the parent's scale transform running alongside it, not a key mismatch or a timing issue.

> Remember to remove this before shipping. The compose lint rules will flag it.
{: .prompt-warning }

## Checklist

If you're implementing cross-screen shared elements in Navigation 3, here's everything that needs to be in place:

1. **`SharedTransitionLayout`** wrapping `NavDisplay`, with the scope passed via `sharedTransitionScope`
2. **`onBack`** provided to `NavDisplay` so it can register its internal back handler
3. **`LocalNavAnimatedContentScope.current`** used as the `animatedVisibilityScope` for `sharedElement`/`sharedBounds` modifiers
4. **`PredictivePopTransitionKey`** overridden with `fadeIn`/`fadeOut` on every route that participates in shared element transitions

## TL;DR

The default predictive back animation in Nav3 scales the entire screen, and that scale transform visually competes with shared elements that are simultaneously animating back to their origin. Override `NavDisplay.PredictivePopTransitionKey` with `ContentTransform(fadeIn(), fadeOut())` on routes that use shared elements. The shared element handles the visual transition; the rest of the screen simply fades away.
