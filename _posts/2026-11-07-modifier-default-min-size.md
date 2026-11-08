---
layout: post
published: true
title: "When \"sp\" is not enough for font scaling"
date: '2026-11-07'
categories: [ Compose, Layout ]
tags: [ android, compose, accessibility, layout ]
image:
  path: /assets/img/banner_default_min_size_compose.png
  alt: Jetpack Compose font scaling with defaultMinSize
---

Most Android devs are aware of using `sp` for text sizes and line heights. There are lint checks which will catch that for you. But sometimes, just using `sp` doesn't make your app scale the text in a legible way.

For most developers, our workflow involves passing the Figma link to the agent, which looks at the specifications and copies it over. Just like the [Android best practices for font scaling](https://developer.android.com/training/multiscreen/screendensities), Figma specifies the font size in `sp` and the component size in `dp`. Under default device settings and short labels, everything looks pixel perfect. However, as soon as you scale the system font settings, let's say to 200%, your UI *might* start to break. The might here is dependent on if you hardcoded the height of the UI component and if the `sp*fontScale` outgrew that height.

## The Example

Let's look at the following button that has a rounded pill shape, a centered 24sp label, and a 48dp height (as probably specified in Figma):

```kotlin
@Composable
fun PillButton(
    text: String,
    modifier: Modifier = Modifier,
) {
    Surface(
        shape = RoundedCornerShape(percent = 50),
        color = Accent,
        modifier = modifier.height(48.dp),
    ) {
        Box(contentAlignment = Alignment.Center) {
            Text(
                text = text,
                color = Color.White,
                fontSize = 24.sp,
                modifier = Modifier.padding(horizontal = 24.dp, vertical = 8.dp),
            )
        }
    }
}
```

At the default font scale (1.0x), the 24sp label and its padding fit comfortably inside the 48dp height:

![PillButton at font scale 1.0](/assets/img/pill_button_scale_100.png){: w="320" }

Now scale the system font size to 200%:

![PillButton clipped at font scale 2.0](/assets/img/pill_button_scale_200_clipped.png){: w="520" }

In Compose, the system font scale is applied through `Density.fontScale` and multiplies every `sp` value, taking our label from 24sp to 48sp.

Meanwhile, `dp` values do not scale with `fontScale`, so the container stays at 48dp. A 48sp label (once you include its font metrics and vertical padding) is taller than 48dp even on a single line. Because `Modifier.height(48.dp)` sets both the minimum and maximum height constraints to `48.dp`, the `Surface` refuses to grow. When `Text` is measured, the available height is locked at 48dp, and the bottom of the text gets clipped right off (slicing the descenders off `y`, `g`, and `p` in this specific example).

## The Fix: `Modifier.defaultMinSize`

To be compatible with font scaling, we need to treat the Figma spec as a minimum height rather than a fixed constraint. This is where `Modifier.defaultMinSize` comes in:


```kotlin
@Composable
fun PillButton(
    text: String,
    modifier: Modifier = Modifier,
) {
    Surface(
        shape = RoundedCornerShape(percent = 50),
        color = Accent,
        modifier = modifier.defaultMinSize(minHeight = 48.dp),
    ) {
        Box(contentAlignment = Alignment.Center) {
            Text(
                text = text,
                color = Color.White,
                fontSize = 24.sp,
                modifier = Modifier.padding(horizontal = 24.dp, vertical = 8.dp),
            )
        }
    }
}
```

At the default 1.0x font scale, the text plus its padding measures within 48dp, so the button looks identical to the `Modifier.height(48.dp)` version. Because both produce the exact same pixels at 1.0x scale, a hardcoded `height` is easy to miss during normal development and code review.

At 2.0x font scale, `defaultMinSize` allows the button to grow vertically to fit the scaled 48sp text:

![PillButton with defaultMinSize at font scale 2.0](/assets/img/pill_button_scale_200_default_min_size.png){: w="520" }

This is also how Material 3's own `Button` is built. It never sets a fixed height; instead, it puts `defaultMinSize` on the `Row` holding its content:

```kotlin
Row(
    Modifier.defaultMinSize(
            minWidth = ButtonDefaults.MinWidth,
            minHeight = ButtonDefaults.MinHeight,
        )
        .padding(contentPadding),
    horizontalArrangement = Arrangement.Center,
    verticalAlignment = Alignment.CenterVertically,
    content = content,
)
```

`ButtonDefaults.MinHeight` acts as a floor, which is why a Material `Button` grows with scaled text instead of clipping.

## How `defaultMinSize` Works Under the Hood

`defaultMinSize` is backed by a single layout node (`UnspecifiedConstraintsNode`), and its `measure` function is simple:

```kotlin
override fun MeasureScope.measure(
    measurable: Measurable,
    constraints: Constraints,
): MeasureResult {
    val wrappedConstraints =
        Constraints(
            if (minWidth.isSpecified && constraints.minWidth == 0) {
                minWidth.roundToPx().fastCoerceIn(0, constraints.maxWidth)
            } else {
                constraints.minWidth
            },
            constraints.maxWidth,
            if (minHeight.isSpecified && constraints.minHeight == 0) {
                minHeight.roundToPx().fastCoerceIn(0, constraints.maxHeight)
            } else {
                constraints.minHeight
            },
            constraints.maxHeight,
        )
    val placeable = measurable.measure(wrappedConstraints)
    return layout(placeable.width, placeable.height) {
        placeable.placeRelative(0, 0)
    }
}
```

Two important details from the measure function above:

1. **`maxWidth` and `maxHeight` are passed through untouched.** The modifier only raises the minimum constraint and doesn't impose a ceiling. This allows the button to grow when `sp * fontScale` outgrows the defined size (48dp in our example).
2. **The `constraints.minHeight == 0` check.** This is the "default" part of `defaultMinSize`. The minimum height is only applied when the incoming minimum constraint is `0`. If a modifier earlier in the chain has already set a non-zero `minHeight`, that constraint wins and `defaultMinSize` doesn't do anything. This makes it perfect for design systems where the caller might want to override the `defaultMinSize` or use percent-based sizing.

## Which Sizing Modifier to Reach For

Nearly all Compose sizing modifiers have two important properties: whether the parent can override the sizing, and whether the child can grow past them:

- **`defaultMinSize(minWidth, minHeight)`**: Sets a minimum only when the incoming minimum is `0`. The parent can override it, and the child can grow past it.
- **`heightIn(min, max)`**: Sets a height range even if the incoming minimum is non-zero (coerced within the parent's constraints). The child can grow up to `max`.
- **`height(x)`**: Sets both `minHeight` and `maxHeight` to `x`. A tighter parent constraint will still win, but the child can never grow past `x`.
- **`requiredHeight(x)`**: Ignores incoming parent constraints and locks both minimum and maximum height to `x`.

A good rule of thumb for design systems is to save fixed modifiers (`size`, `height`) for intrinsic dimensions that don't depend on text, like a 24dp icon or a 1dp divider. Anything involving text should use `defaultMinSize` or `heightIn` so the text can scale without being clipped.

## Testing

Now that we've fixed the issue, how can we prevent it from happening again?

### 1. `@PreviewFontScale`

We can instruct agents (through Agents.md or skills) to always generate preview variants and look at them. `@Preview` accepts a `fontScale` parameter, and `@PreviewFontScale` expands a single preview into seven font scales (from 85% up to 200%):

```kotlin
@PreviewFontScale
@Composable
private fun PillButtonPreview() {
    AppTheme {
        PillButton(text = "Create your first group")
    }
}
```

### 2. Compose UI Tests

If your previews feed into screenshot tests (like Roborazzi or Paparazzi), the 200% preview will catch a fixed-height regression visually. You can also test the behavior directly with `createComposeRule()` by driving `fontScale` from state and asserting that the button's height increases:

```kotlin
@get:Rule
val composeRule = createComposeRule()

private fun measureAtFontScales(
    button: @Composable (Modifier) -> Unit,
): Map<Float, Dp> {
    val fontScale = mutableFloatStateOf(1f)

    composeRule.setContent {
        val base = LocalDensity.current
        CompositionLocalProvider(
            LocalDensity provides Density(base.density, fontScale = fontScale.value)
        ) {
            button(Modifier.testTag("pill"))
        }
    }

    val heights = mutableMapOf<Float, Dp>()
    listOf(1f, 2f).forEach { scale ->
        composeRule.runOnIdle { fontScale.value = scale }
        composeRule.waitForIdle()
        heights[scale] = composeRule.onNodeWithTag("pill")
            .fetchSemanticsNode()
            .let { with(composeRule.density) { it.size.height.toDp() } }
    }
    return heights
}

@Test
fun minSizeButtonGrowsWithFontScale() {
    val heights = measureAtFontScales { PillButton("Create your first group", it) }

    assertThat(heights.getValue(1f)).isEqualTo(48.dp)
    assertThat(heights.getValue(2f)).isGreaterThan(heights.getValue(1f))
}
```

> `fontScale` is updated via `mutableFloatStateOf` inside a single `setContent` block because `createComposeRule()` only allows calling `setContent` once per test.
{: .prompt-info }


## TL;DR

Using `sp` for text sizes is only half the story for font scaling. When an agent (or developer) translates a fixed component height from Figma into `Modifier.height(48.dp)`, the container is locked at 48dp even when `sp * fontScale` outgrows that height, clipping the label. Use `Modifier.defaultMinSize(minHeight = 48.dp)` for components that render text, so the spec height acts as a floor at default scale while allowing the component to grow when system font size increases.
