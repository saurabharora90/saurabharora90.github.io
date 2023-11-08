---
layout: post
published: true
title: Navigating Pitfalls - When to Use derivedStateOf with remember(key) in Jetpack Compose
date: '2023-11-09'
categories: Compose
tags: [android, compose]
image:
  path: /assets/img/banner_compose.png
  alt: Jetpack Compose
---

Jetpack Compose has transformed Android UI development by providing a straightforward approach to building user interfaces. One of its core principles is minimising unnecessary updates, and one of the tools it offers is `derivedStateOf`. In this blog post, we will explore the effective utilisation of `derivedStateOf` through an example of hashtag validation, as well as highlighting some potential pitfalls we encountered.

First, let's grasp the concept of `derivedStateOf`. This Composable function is useful when you have a state or a key that changes frequently, but you want to optimise the recomposition of your UI to occur only when specific conditions are met.

For a deeper dive into `derivedStateOf`, I recommend reading Ben Trengrove's excellent [blog post](https://medium.com/androiddevelopers/jetpack-compose-when-should-i-use-derivedstateof-63ce7954c11b).

## The Hashtag Validation Example

Imagine a scenario where you allow users to add hashtags to a post. You have a Composable `PostHashtags` that looks like this:

<script src="https://gist.github.com/saurabharora90/f1fc38b795020f0f8768fb9eb923f3e8.js"></script>

In this code, we allow users to input hashtags, and an error message is displayed when an invalid hashtag is entered. Additionally, we use `hashtags` as the remember key for `inputHashTag`. This ensures that the input field is cleared once a hashtag is successfully submitted.

![Kapture 2023-10-29 at 23.41.42.gif](/assets/img/Kapture_2023-10-29_at_23.41.42.gif){: w="320" h="4" }


> One could argue that onClick should be responsible for clearing the mutable state instead of relying on the side effect of recreating the hashtags list. While this is true, there are other considerations to take into account before clearing the state, such as ensuring that the server accepts the hashtags and validating that there are no duplicates, among other things.
{: .prompt-warning }

Now, we also want to check whether the entered hashtag is valid and display an error if it's not. The ideal scenario here is to recompose the error text only when the hashtag changes from valid to invalid or vice versa, not with every keystroke. This is the perfect use case for `derivedStateOf`.

<script src="https://gist.github.com/saurabharora90/d38eba197fc8cad51b4ad715e5456d67.js"></script>

![Kapture 2023-10-29 at 23.50.45.gif](/assets/img/Kapture_2023-10-29_at_23.50.45.gif){: w="320" h="4" }

With this setup, unnecessary recompositions of the error text label is eliminated, and the error text appears only when the hashtag contains a space.

## The Unexpected Behaviour

But there's an issue. After adding the first hashtag, the error text and error state on the text field are no longer shown.

![Kapture 2023-10-29 at 23.53.05.gif](/assets/img/Kapture_2023-10-29_at_23.53.05.gif){: w="320" h="4" }

To investigate, let’s add some log statements:

<script src="https://gist.github.com/saurabharora90/ccd6423db4d5f719d83797b24641da11.js"></script>

Surprisingly, the log statements show that the `derivedStateOf` calculation block is no longer called after adding the first hashtag.

![Screenshot 2023-10-29 at 9.24.40 PM.png](/assets/img/Screenshot_2023-10-29_at_9.24.40_PM.png)

## The Root Cause

The root cause of this issue stems from the behaviour of the `remember` function. When the `hashtags` list changes, it generates and remembers a new value for `inputHashTag`. However, this "new value" is not captured by the `derivedStateOf` function, and it continues to work with the "old" `inputHashTag` state object. In other words, when the "new" `inputHashTag` state object updates, the calculation block of `derivedStateOf` does not execute because it observing changes on the outdated `inputHashTag` state object.

## The Solution

There are two possible options to resolve this issue:

1) We can ensure that `derivedStateOf` captures the new instance of `inputHashTag` whenever it changes. We can achieve this by adding `hashtags` as a key to the `remember` of `isValidHashtag`. This forces `remember` to produce a new derived state object whenever `hashtags` are updated (just like it does for `inputHashTag`). This guarantees that the updated `inputHashTag` is captured and utilised by the new `derivedStateOf` object.

<script src="https://gist.github.com/saurabharora90/d6e05be8840b8c5e72e8fff9d6bd5aa5.js"></script>

2) Another approach would be to key the `remember` block with the `inputHashTag` **state object**. This change involves not relying on the `by` property delegate to directly access the value of `inputHashTag`. Instead, we've opted to work with the state object itself. By keying the `remember` block to the state object, we ensure that it recomposes every time a new state object is created. This guarantees that the updated `inputHashTag` is captured and utilised by the new `derivedStateOf` object.


> Note: We should not key the `remember` block to the `inputHashTag` **value**. In such a scenario, as the user inputs hashtags, the `inputHashTag` value would change for each keystroke. This would result in the `remember` block running repeatedly, causing unnecessary recomposition.
{: .prompt-danger }

<script src="https://gist.github.com/saurabharora90/e3aba869495c4349166a7dd304c40e86.js"></script>

With either of this approach, the bug is resolved, and the `derivedStateOf` functions as we hoped it would.

![Kapture 2023-10-30 at 00.03.29.gif](/assets/img/Kapture_2023-10-30_at_00.03.29.gif){: w="320" h="4" }

## Conclusion

`derivedStateOf` can be a bit tricky to use, but it's an extremely effective tool in Jetpack Compose for reducing unnecessary recompositions. It enables you to control when and how your UI updates, ensuring a smooth and efficient user experience. When applied thoughtfully, it can lead to more performant and responsive applications.

So, the next time you're dealing with dynamic UI components and want to optimize recompositions, consider incorporating `derivedStateOf` into your Jetpack Compose arsenal. Happy composing!

------------------------------------------------------------------------------------------------------------------------

<sup>Big thanks to [Ben Trengrove](https://twitter.com/bentrengrove?lang=en) for the review</sup>