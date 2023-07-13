---
layout: post
title: Cleaner pull requests for Kotlin & Java code
subtitle: Code styling and formatting using Checkstyle and ktlint
published: true
date: '2020-02-26'
tags: [android, code formatting]
---
Recently I looked into reducing the noise from our pull requests. The bulk of the effort involved setting up a code styling guide, formatting all our code based on the style guide and setting up automated checks to capture formatting errors. Having a properly formatted code ensures pull requests don't have noise in the form of whitespaces, function ordering, braces, etc and rather focuses on the enhancements.

Through this post, I would like to detail some of my learnings and how other developers can setup these checks on their projects.

### Choosing a Code Style

Our entire team wasn't picky about what code style we chose. The only consideration we had was it should require minimal setup effort and should be easy for a new developer to adopt. Since our codebase is a mixture of Java and Kotlin code, we needed guides and tools for both.

### Code Style and Formatting Java Code

For Java, the process was really straight forward. We decided to stick with the IDE defaults without adding any customization to it.

We used [Checkstyle](https://checkstyle.sourceforge.io/) to help capture any formatting errors.

>Checkstyle is a development tool to help programmers write Java code that adheres to a coding standard. It automates the process of checking Java code to spare humans of this boring (but important) task. This makes it ideal for projects that want to enforce a coding standard.

It needs a configuration file that defines the formatting "rules". We [created](https://gist.github.com/saurabharora90/039ba64aecef47148ec19e0d98722c17) one which closely matches the formatting style of the IDE. This was heavily borrowed from the [Google Checks](https://github.com/checkstyle/checkstyle/blob/master/src/main/resources/google_checks.xml) configuration file.

To validate the setup, we first formatted our entire codebase by using the IDE defaults. Make sure you are using the IDE Defaults for Java and not a modified version. The default can be reset via the IDE settings/preferences under Editor->Code Style->Java->Set from Predefined style -> Android). 

To format your entire codebase, open your project in Android Studio, right-click on the root folder and select "Reformat Code". Make sure the File mask is set to Java files.

![Format-Java](/assets/img/reformat_java.png)

Once the IDE is done formatting, you can run Checkstyle on your codebase to verify if there are any unresolved issues. If this is the first time that you are formatting your codebase (as was the case for us), you might have some leftover issues which the IDE couldn't auto-format. You can try to fix them one by one or leave these to be addressed when future changes are made to these files. We did leave a few files like that since they had a lot of formatting errors.

*Pro-tip:* You can install a [Checkstyle plugin](https://plugins.jetbrains.com/plugin/1065-checkstyle-idea) which makes it easy to run Checkstyle on a single file, entire module or an entire project. You can also load your own rules configuration file through this plugin.

*Note:* Checkstyle does not auto-format the code to resolve formatting issues. It will just highlight the issues in the codebase.

### Code Style and Formatting Kotlin Code

When it came to Kotlin, I realized that there are two popular Kotlin Style guides, one by [Jetbrains](https://kotlinlang.org/docs/reference/coding-conventions.html) and another on the [Android developer site](https://developer.android.com/kotlin/style-guide). Turns out, Android Studio by default uses the Kotlin Coding conventions (I was a bit surprised that it wasn't based off the Android Developer guide but then Android Studio is based off IntelliJ, who also maintain the Kotlin Plugin, so it probably makes sense that the default is Jetbrains Style?). 

To help in catching any formatting errors, we decided to use [ktlint](https://github.com/pinterest/ktlint).

>An anti-bikeshedding Kotlin linter with built-in formatter 

ktlint is an opinionated Kotlin code formatting tool. Unlike Checkstyle, ktlint is also able to auto-format and fix common formatting errors. 

Unfortunately, out of the box ktlint didn't work with the default IDE settings. The conflict between ktlint and IDE defaults lay in wildcard imports (`import java.util.*` vs `import java.util.Date`) and import ordering. ktlint is opinionated about both Wildcard imports (it discourages them) and prefers Alphabetical import ordering. However, the Kotlin Coding Convention does not mention anything about Wildcard imports or how imports should be ordered. On the other hand, the Android Style guide is [opininated](https://developer.android.com/kotlin/style-guide#structure) about them.


Here, we felt we should use the Android Style guide as our default code style if there is a way to configure Android Studio with all those settings. Turn out, ktlint offers a `--android` switch to configure ktlint to use the Android Kotlin Guide. It even offers a way to install this code style in your IDE and claims (as of release 0.36.0) "it makes Intellij IDEA's built-in formatter produce 100% ktlint-compatible code." In practice, this didn't seem to work. Turns out the Kotlin Gradle Plugin [does not allow customization](https://youtrack.jetbrains.com/issue/KT-10974) of import order so ktlint will continue to complain about import ordering and no one has the time to manually sort the imports in alphabetical order.

With this limitation in mind and no evident advantage of using the Android Style, we decided to switch back and stick with the IDE default (Editor->Code Style->Kotlin->Set from Predefined style -> Kotlin Style Guide) and eliminate the overhead of setting a custom style for our project. However, we still have the issue of both Wildcard imports and Alphabetical import ordering as ktlint is opinionated about them (even without the `--android` switch).

Given the above and our desire to have a setup which needs the least overhead, we started exploring if we could disable these conflicting rules. Enter [.editorconfig](https://editorconfig.org/) files.

>EditorConfig project consists of a file format for defining coding styles 

Both Android Studio (and other [IntelliJ](https://blog.jetbrains.com/idea/2019/06/managing-code-style-on-a-directory-level-with-editorconfig/) products) and ktlint respect .editorconfig files. As these files can also be checked into VCS, it opened the door for us to disable these offending rules and have a zero overheard setup for code styling.

In our .editorconfig file (at the root level of your project), we added:

```
"disabled_rules = import-ordering,no-wildcard-imports"
```

The Kotlin Coding Conventions doesn't mention anything about imports; the IDE does not support import ordering, so our editor config files instructed ktlint to disable those rules.

We finally had a zero-overhead setup for ktlint for our project. No need to modify the IDE settings, just clone the project repo and you are on your way.

To validate the setup, we first formatted our entire codebase by using the IDE defaults. To format your entire codebase, open your project in Android Studio, right-click on the root folder and select "Reformat Code". This time make sure the File mask is set to Kotlin files. Once the IDE is done formatting, you can run ktlint on your codebase to verify if there are any unresolved issues (hopefully there should be none or a few that the IDE couldn't auto format).

*P.S.* We could choose to not disable wildcard imports but then that would need us to modify the IDE default settings for Kotlin and not give us a zero-overhead setup.
*P.P.S.* IntelliJ/Studio still does import ordering just that it doesn't do it alphabetically for Kotlin files (which is what ktLint expects). So you do get import ordering, just a different one. Also, new imports are added to the correct place by the IDE.
*P.P.P.S* You can choose to setup up a custom code style, add it to your VCS and instruct Android Studio to load it by default. Even that will give a zero-overheard setup. We just didn't want to have a custom code style and the entire team was aligned on that.

### Automate Checking

We explored two options for automating these checks for developers. We could either (1) use ktlint and Checkstyle Gradle Plugins add this to our CI or (2) use git hooks. We eventually decided to use git hooks because we felt this provided a faster turnaround and made sure that PRs coming in were already formatted, rather than relying on our CI to let us know if it wasn't.

We added pre-commit hooks for both checkstyle and ktlint, which would check the files being committed and run checks on them. Staying with our mantra of least configuration, the git hooks are checked into VCS into a `.githooks` folder. By default, githooks get added to `.git/hooks` folder but the `.git` folder is not checked into VCS. Hence we decided to change the [defaults](https://git-scm.com/docs/githooks). Once someone clones the repository, they need to run:

```
git config core.hooksPath .githooks
```

to setup git to use the hooks from the `.githooks` folder.



That's it! If you have any questions or suggestions, leave a comment below or hit me up on [Twitter](https://twitter.com/saurabh_arora90).

##### Random musings:

- The developer community is really passionate about whether we should use tabs or spaces. All the discussions made for a fascinating read.
- Apparently using spaces can make you [richer](https://stackoverflow.blog/2017/06/15/developers-use-spaces-make-money-use-tabs/)
- A Googler analyzed 40,000 repositories for tabs vs spaces and concluded the [winner](https://www.reddit.com/r/programming/comments/50f5r9/400000_github_repositories_1_billion_files_14/)
- Set up a macro to auto-format your code file once you are done with it. This will help reduce the errors that are flagged by Checkstyle and ktlint. See [here](https://twitter.com/saurabh_arora90/status/1075305267003158528) for how to set up a macro
- I have only highlighted the import ordering difference between Kotlin Coding Conventions and Android Kotlin Style Guide. There are other differences as well, which warrant a `--android` switch for ktlint.
- [Github issue](https://github.com/pinterest/ktlint/issues/527) documenting ktlint's Alphabetical import ordering being a compatibility issue with Android Studio