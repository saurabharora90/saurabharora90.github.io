---
layout: post
published: true
title: Curious case of missing Bottom Sheet Previews
date: '2025-07-03'
categories: Compose Bottom-Sheet
tags: [android, compose]
image:
  path: /assets/img/banner_compose.png
  alt: Jetpack Compose
---

While working on a Jetpack Compose project, I noticed my modal bottom sheet wasn’t appearing in Android Studio previews. I could preview the sheet’s contents by extracting them into a separate composable, but the full bottom sheet, including its scrim and drag handle, was invisible. This was an issue because I needed to verify the entire UI for design and screenshot tests.

## Why Full Previews Matter

Previewing the entire bottom sheet is essential to verify:

- **Scrim Color and Opacity**: Ensures the overlay behind the sheet has the correct appearance.
- **Drag Handle Appearance**: Confirms the drag handle is styled and positioned correctly.
- **Overlay Behavior**: Verifies how the sheet appears and how the content being overlaid looks

Full functioning previews also mostly imply compatibility with screenshot tests  

## The Issue

The problem stems from `rememberModalBottomSheetState`, which initializes the bottom sheet to a `Hidden` state. The `ModalBottomSheet` composable uses a `LaunchedEffect` to call `sheetState.show()` and make the sheet visible:

```kotlin
@Composable
@ExperimentalMaterial3Api
fun ModalBottomSheet(
    onDismissRequest: () -> Unit,
    modifier: Modifier = Modifier,
    sheetState: SheetState = rememberModalBottomSheetState(),
    // ... other parameters ...
) {
    // ... implementation ...
    if (sheetState.hasExpandedState) {
        LaunchedEffect(sheetState) { sheetState.show() }
    }
    // ... rest of the implementation ...
}

```

Since previews don’t execute `LaunchedEffect`, the sheet remains hidden. The `rememberModalBottomSheetState` function sets the initial state to `Hidden` and doesn’t allow customization:

```kotlin
@Composable
@ExperimentalMaterial3Api
fun rememberModalBottomSheetState(
    skipPartiallyExpanded: Boolean = false,
    confirmValueChange: (SheetValue) -> Boolean = { true }
) = rememberSheetState(
    skipPartiallyExpanded = skipPartiallyExpanded,
    confirmValueChange = confirmValueChange,
    initialValue = Hidden
)

```

The `rememberSheetState` function it calls is internal, so we can’t modify the initial state directly:

```kotlin
@Composable
@ExperimentalMaterial3Api
internal fun rememberSheetState(
    skipPartiallyExpanded: Boolean = false,
    confirmValueChange: (SheetValue) -> Boolean = { true },
    initialValue: SheetValue = Hidden,
    skipHiddenState: Boolean = false,
    positionalThreshold: Dp = BottomSheetDefaults.PositionalThreshold,
    velocityThreshold: Dp = BottomSheetDefaults.VelocityThreshold,
): SheetState

```

## The Solution

For previews, use `rememberStandardBottomSheetState`, which initializes the sheet to a `PartiallyExpanded` state, making it visible without relying on side effects:

```kotlin
@Composable
@ExperimentalMaterial3Api
fun rememberStandardBottomSheetState(
    initialValue: SheetValue = PartiallyExpanded,
    confirmValueChange: (SheetValue) -> Boolean = { true },
    skipHiddenState: Boolean = true,
) = rememberSheetState(
    confirmValueChange = confirmValueChange,
    initialValue = initialValue,
    skipHiddenState = skipHiddenState
)

```

> If your sheet is tall, the you can specify the initialValue as `SheetValue.Expanded`  to get the full sheet in preview.
{: .prompt-info }

So instead of relying on separating the contents of the bottom sheet to its own composable, just for previews, we can structure the composable to accept a `sheetState` parameter, defaulting to `rememberModalBottomSheetState` for production, but using `rememberStandardBottomSheetState` in previews:

```kotlin
@Composable
fun MyBottomSheet(
    onDismiss: () -> Unit,
    sheetState: SheetState = rememberModalBottomSheetState()
) {
    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState,
        // other parameters...
    ) {
        // content...
    }
}

@Preview
@Composable
fun MyBottomSheetPreview() {
    MyBottomSheet(
        onDismiss = {},
        sheetState = rememberStandardBottomSheetState()
    )
}

```

Alternatively, you could create a custom `SheetState` object:

```kotlin
class SheetState(
    internal val skipPartiallyExpanded: Boolean,
    positionalThreshold: () -> Float,
    velocityThreshold: () -> Float,
    initialValue: SheetValue = Hidden,
    internal val confirmValueChange: (SheetValue) -> Boolean = { true },
    internal val skipHiddenState: Boolean = false,
)

```

However, `rememberStandardBottomSheetState` is simpler (personal opinion) and sufficient for previews.

## Why Not Use `rememberStandardBottomSheetState` in Production?

Using `rememberStandardBottomSheetState` in production is problematic because it starts the sheet in a `PartiallyExpanded` state, making it visible immediately. Modal bottom sheets should start hidden and animate in for a better UX. `rememberModalBottomSheetState` ensures this by initializing the sheet to `Hidden` and then animating the sheet in via `sheetState.show()`.

## Conclusion

To render bottom sheet previews in Jetpack Compose, use `rememberStandardBottomSheetState` to make the sheet visible in Android Studio previews and screenshot tests. In production, use `rememberModalBottomSheetState` for a hidden initial state and smooth animations. This approach ensures accurate design verification and reliable screenshot tests, capturing the full UI as users will experience it.