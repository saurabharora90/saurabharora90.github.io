---
layout: post
published: false
title: "Giving Every LazyColumn Row Its Own ViewModel"
date: '2026-04-28'
categories: [Compose, Android, ViewModel]
tags: [android, compose, viewmodel, navigation3, lifecycle]
image:
  path: /assets/img/banner_compose.png
  alt: Jetpack Compose
---

A `ViewModel` per screen is standard. A `ViewModel` per *row* has not had an official story. Teams have invented their own approaches which are proprietary to their codebases.

Lifecycle 2.11's alpha releases close that gap. `rememberViewModelStoreOwner()` gives Compose a first-party way to scope `ViewModel`s to a `LazyColumn` row. Paired with `SavedStateHandle` and Nav3's per-entry decorator, per-row state survives scroll-off, configuration changes, and process death without custom infrastructure.

## The Example

The example we’ll use here, is a feed of posts rendered in a `LazyColumn`. Each row shows an avatar, an author, body text with a Show more affordance, and a row of actions (favorite, comment, bookmark, share).

```kotlin
class HomeViewModel : ViewModel() {
    private val _posts = MutableStateFlow<List<Post>>(emptyList())
    val posts: StateFlow<List<Post>> = _posts.asStateFlow()

    fun toggleFavorite(post: Post) {
        _posts.value = _posts.value.map {
            if (it.id == post.id) it.copy(isFavorite = !it.isFavorite) else it
        }
        viewModelScope.launch {
            // simulate API call
            delay(2.seconds)
        }
    }

    fun toggleBookmark(post: Post) { /* optimistic, rollback on failure */ }
}
```

And the screen passes callbacks down through the list:

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    val posts by viewModel.posts.collectAsState()
    LazyColumn {
        items(posts, key = { it.id }) { post ->
            PostItem(
                post = post,
                onFavoriteClick = { viewModel.toggleFavorite(it) },
                onBookmarkClick = { viewModel.toggleBookmark(it) },
            )
        }
    }
}
```

![simple_start_sample.gif](/assets/img/simple_start_sample.gif){: w="320" h="4" }

For a demo-sized feed, this works.

## Why one feed ViewModel isn't enough

A real feed is rarely a list of one item type. It will typically carry `Post`, `AdItem`, `SuggestedUserCarousel`, maybe a `PollPost`. Each has its own data sources, its own interactions, its own failure modes, and its own telemetry. An `AdItem` needs impression tracking and clickthrough reporting. A `PollPost` needs vote submission with rollback.

Put all of that in `HomeViewModel` and it becomes the union of every child type's logic. Every new row type adds methods, state fields, and a bug surface that can break unrelated rows. A bookmark rollback failing on a `Post` should have no way to reach into a `PollPost` three rows down.

A cleaner model is one `ViewModel` per item *type*, scoped per item *instance*. `PostViewModel` owns post behavior. `PollViewModel` owns polls. `AdViewModel` owns impression logic. `HomeViewModel` goes back to sourcing the list and telling the UI what to render.

> In the sample I'm keeping things simple: only one item type, and `PostViewModel` is a straight 1:1 map from `Post` to `PostItemUiState`. In real code the per-item VM is where you'd compute relative timestamps, merge reactive per-user signals like is-liked-by-me, and sequence async behavior. The simplification here keeps the focus on the scoping mechanics.
{: .prompt-info }

## A Quick Primer on `viewModel()`

> If you already know how `viewModel()` resolves its instance and what a `ViewModelStoreOwner` is, you can skip this section.
{: .prompt-info }

A **`ViewModelStore`** is a map from key to `ViewModel`. A **`ViewModelStoreOwner`** is anything that holds one such map: an Activity, a Fragment, a Nav3 `NavEntry`. When you call `viewModel()` in a composable, it reads `LocalViewModelStoreOwner.current`, asks that owner's store for a `ViewModel` of the requested type, and returns the existing instance or creates a new one via the factory. When the owner is destroyed, the store clears, calling `onCleared()` on every `ViewModel` inside.

In a Nav3 app with `rememberViewModelStoreNavEntryDecorator()` installed, `LocalViewModelStoreOwner.current` inside an entry resolves to a `ViewModelStoreOwner` created for that entry. Pop the entry, its store clears, its VMs clear with it. This is the per-screen ViewModel guarantee we are all used to.

The [official ViewModel docs](https://developer.android.com/topic/libraries/architecture/viewmodel) cover the lifecycle guarantees in more detail. What matters here: **the owner that `viewModel()` sees is the one installed by the nearest `LocalViewModelStoreOwner`, and everything in its store shares that owner's lifetime.**

## The common first pass: `viewModel()` in the row

A reasonable first move is to push `viewModel()` down into the row composable:

```kotlin
@Composable
fun PostItem(post: Post) {
    val viewModel: PostViewModel = viewModel(factory = PostViewModel.Factory(post))
    /* ... */
}
```

This does not do what it looks like it does.

`viewModel()` resolves against `LocalViewModelStoreOwner.current`, which inside a `LazyColumn` row is still the nav entry's owner. Without a key, the store keys the VM by class. Every row asks for "the `PostViewModel` in this store," and the store has exactly one. The factory runs for the first row that composes, and every row after that gets the same instance back, still holding the `post` that seeded the first call. Twelve rows, one `PostViewModel`.

Adding a key fixes the multiplicity:

```kotlin
val vm: PostViewModel = viewModel(
    key = post.id.toString(),
    factory = PostViewModel.Factory(post),
)
```

Now each row gets its own instance. But those instances still live in the nav entry's store, which only clears when the screen is popped. Scroll the list once end to end and you've accumulated a VM per post you've ever looked at, with no way to evict any single one. This may be fine for your use case, but for heavier VMs it can cause memory issues (your mileage may vary).

What we actually need is each row to have its own store, with the row's lifecycle determining when it's cleared.

## Scoping per row

`rememberViewModelStoreOwner()` is new in Lifecycle 2.11. It returns a `ViewModelStoreOwner` scoped to the call site. The owner is created when the composable enters composition and is cleared, alongside every `ViewModel` it holds, when the composable leaves.

The parent it links to is the nearest `LocalViewModelStoreOwner` in the composition. In a Nav3 app that's the nav entry's owner, so configuration changes (rotation) and process recreation are handled correctly: child VMs survive rotation if the parent does, and are destroyed when the parent is destroyed.

The child owner inherits the parent's `SavedStateRegistryOwner`, default `ViewModelProvider.Factory`, and `CreationExtras`. So `SavedStateHandle`, Hilt injection, and application-level defaults keep working with no additional wiring.

Here's how it slots into `HomeScreen`:

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    val posts by viewModel.posts.collectAsState()

    LazyColumn {
        items(posts, key = { it.id }) { post ->
            val storeOwner = rememberViewModelStoreOwner()
            CompositionLocalProvider(LocalViewModelStoreOwner provides storeOwner) {
                PostItem(post = post)
            }
        }
    }
}
```

Two things happen here:

1. For each row, `rememberViewModelStoreOwner()` mints an owner scoped to that row's call site.
2. We swap `LocalViewModelStoreOwner` to that owner for the row's content. `PostItem`'s unchanged `viewModel()` call now resolves against the row's own store.

Here we have `PostViewModel` log on init and clear so we can watch the scoping behavior from the outside:

```kotlin
class PostViewModel(
    private val post: Post,
    private val savedStateHandle: SavedStateHandle,
) : ViewModel() {
    init { Log.i("Saurabh", "Post init: ${post.authorName}") }
    override fun onCleared() { Log.i("Saurabh", "Post Cleared: ${post.authorName}") }
    /* state, toggles, expand */
}
```

Watch the logs as we scroll: each row's VM initializes when it enters composition and clears when it scrolls past.

![Logcat.gif](/assets/img/Logcat.gif)

Each row has an independent `ViewModel`. `viewModelScope` for launching coroutines is also now scoped per the lifecycle of the row. If you have polling logic in the VM, it automatically cancels when the row is disposed.

The parent `LocalViewModelStoreOwner` that the row's owner links to can be anything: an Activity, a Fragment, a Nav3 nav entry. In a Nav3 app, `rememberViewModelStoreNavEntryDecorator()` installs a per-entry owner, so every row's owner gets cleared when that entry is popped. Nothing leaks across navigation.

## Restoring ViewModel state

With the scoping in place, we can move the Show more state out of the composable and into the VM, where the rest of the row's state already lives:

```kotlin
class PostViewModel(private val post: Post, /* ... */) : ViewModel() {
    private val _state = MutableStateFlow(
        PostItemUiState(/* ..., */ isExpanded = false)
    )
    val state: StateFlow<PostItemUiState> = _state.asStateFlow()

    fun expand() { _state.update { it.copy(isExpanded = true) } }
}
```

The composable reads `state.isExpanded` and calls `viewModel::expand` on tap, replacing the local `remember { mutableStateOf(false) }`.

This looks like a tidy consolidation, but it doesn't survive scroll-off. Expand a row, scroll it off-screen, scroll back, and it's collapsed again.

![broken_collapse.gif](/assets/img/broken_collapse.gif){: w="320" h="4" }

The behavior matches the raw `remember { mutableStateOf }` version, since that version wasn't `rememberSaveable` either. The difference is that the ViewModel was supposed to be the durable state holder for the row, and it isn't durable enough. Scroll the row off, the row leaves composition, the row's owner is cleared and the VM is gone. When the row scrolls back, a fresh owner mints a fresh VM with `isExpanded = false`.

The VM alone can't fix this. The VM is the thing being thrown away. The answer is to persist `isExpanded` *outside* the VM's lifetime, in a store that lives longer than the row's visit to the viewport but is still scoped to that row. `SavedStateHandle`, backed by a per-item `SaveableStateHolder` slot, is exactly that.

`SavedStateHandle` is a key-value bundle scoped to a `ViewModel`. Reads and writes flow through the underlying `SavedStateRegistry`, which means the values survive process death too. Its lifetime is tied to the nearest `SavedStateRegistryOwner`, which for our scoped child owner is inherited from the parent (the nav entry).

We pull one in via `CreationExtras.createSavedStateHandle()` in the factory, which works because the new scoped owner propagates the parent's `CreationExtras`:

```kotlin
class PostViewModel(
    private val post: Post,
    private val savedStateHandle: SavedStateHandle,
) : ViewModel() {

    companion object {
        fun Factory(post: Post) = viewModelFactory {
            initializer { PostViewModel(post, createSavedStateHandle()) }
        }
    }

    private val _state = MutableStateFlow(
        PostItemUiState(
            /* ..., */
            isExpanded = savedStateHandle["isExpanded"] ?: false,
        )
    )
    val state: StateFlow<PostItemUiState> = _state.asStateFlow()

    fun expand() {
        _state.update { it.copy(isExpanded = true) }
        savedStateHandle["isExpanded"] = true
    }
}
```

On its own, this isn't enough. All twelve rows' `SavedStateHandle`s resolve against the same parent `SavedStateRegistry` (the nav entry's). If every row writes to the key `"isExpanded"`, they overwrite each other. Row 1 expanding would flip every row's `isExpanded` the next time it composed.

`SaveableStateHolder.SaveableStateProvider(key)` fixes this. It installs a nested saved-state slice keyed by whatever you hand it, so anything saveable inside (including a child `SavedStateRegistry`, and therefore our `SavedStateHandle`) gets its own bucket. Pair it with the scoped `ViewModelStoreOwner`:

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    val saveableStateHolder = rememberSaveableStateHolder()
    val posts by viewModel.posts.collectAsState()

    LazyColumn {
        items(posts, key = { it.id }) { post ->
            saveableStateHolder.SaveableStateProvider(post.id) {
                val storeOwner = rememberViewModelStoreOwner()
                CompositionLocalProvider(LocalViewModelStoreOwner provides storeOwner) {
                    PostItem(post = post)
                }
            }
        }
    }
}
```

The flow is now:

1. Row 1 expands. `PostViewModel` writes `isExpanded = true` to its `SavedStateHandle`.
2. Row 1 scrolls off. The `SaveableStateProvider` leaves composition and snapshots everything saveable inside it (including the `SavedStateRegistry` backing our `SavedStateHandle`) into its per-key bucket. The row's `ViewModelStoreOwner` is also cleared and `PostViewModel.onCleared()` runs. The snapshot outlives the VM.
3. Row 1 scrolls back. `SaveableStateProvider(post.id)` restores the bucket. A fresh `ViewModelStoreOwner` is minted, a fresh `PostViewModel` is created, and its new `SavedStateHandle` reads from the restored slice. `isExpanded` comes back as `true`, and row 1 renders expanded.

![final_fixed.gif](/assets/img/final_fixed.gif){: w="320" h="4" }

Process-death survival comes for free. The nav entry's `SavedStateRegistry` is persisted by Nav3, and the per-item slices go with it.

## Checklist

If you're reaching for per-item `ViewModel`s in Compose:

1. **Hoist** `rememberSaveableStateHolder()` above the list.
2. **Wrap** each item in `saveableStateHolder.SaveableStateProvider(key) { ... }` so per-item `SavedStateHandle` values don't collide.
3. **Mint** a scoped `ViewModelStoreOwner` with `rememberViewModelStoreOwner()` and provide it via `LocalViewModelStoreOwner`. The owner (and its VMs) is cleared automatically when the item leaves composition.

## TL;DR

Lifecycle 2.11's alpha `rememberViewModelStoreOwner()` lets every row in a `LazyColumn` have its own `ViewModel`, scoped to the row's call site, cleaned up automatically when the row leaves composition. The child owner inherits the parent's factory, `CreationExtras`, and `SavedStateRegistryOwner`, so `SavedStateHandle` and injection keep working. Pair the owner with a per-item `SaveableStateHolder.SaveableStateProvider` slot so per-row state in `SavedStateHandle` survives scroll-off and process death.

<sup>The approach in this post leans on Marcello Galhardo's [Scoping ViewModels in Compose](https://marcellogalhardo.dev/posts/scoping-viewmodels-in-compose/), which is the clearest writeup of the motivation for these APIs.</sup>
