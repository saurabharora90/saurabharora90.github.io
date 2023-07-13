---
layout: post
published: true
title: Proguard/R8 in the world of modularity
date: '2020-01-10'
tags: [android, modular]
---
The benefits of modularing our apps have been echoed a lot in the community and by Google itself. Here I would like to discuss how we should look into handling Proguard/R8 rules files in this new world.

Generally, developers tend to maintain their Proguard rules file in their application module. This is mostly okay if the modules are only consumed by one application. But what if those modules are shared across multiple apps? Like at Viki, where most of our modules have code that is used in both TV and mobile apps. With shared modules across apps, changes in our module code and missing Proguard configuration would sometimes break our release builds for both mobile and TV apps. Initially we would add the missing proguard rules to both the apps but eventually, we felt it might serve us better to have the proguard rules in the module directly. 

By including the proguard rules in the modules directly, the consuming apps don't need to worry about updating their rules because a module that they include has changed its internal dependencies or structures.

### Consumer Proguard Files

If we use Android Studio to create a new module (Android Library), then the `build.gradle` file that is generated for the new module, contains a `consumerProguardFiles 'consumer-rules.pro'` section under `defaultConfig`. In case you already have an existing module, you can add a section for the `consumerProguardFiles` as follows:

```groovy
	defaultConfig {
        minSdkVersion 21
        targetSdkVersion 29
        
        .....
        
        consumerProguardFiles 'consumer-rules.pro'
    }
```

A consumer proguard rules file is like any other proguard rules files with the caveat that the rules inside it are automatically applied to the consuming application when the application is being built in a proguard enabled mode. This file is not used for immediate minification (i.e. when building the library itself). It is only used when the consumer of the library is being built. We should anyways not be minifying the module when it is being built, because at that point, the module doesn't know what parts of it are being consumed. It is for this reason that the default `build.gradle` that is generated for a library module has `minifyEnabled` as `false`

```groovy
buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
```

### Adding Rules

To bundle proguard rules with your module, add the rules to the `consumer-rules.pro` file. At Viki, we have a module that houses all our playback logic (including ads) and we always add the ad provider's Proguard rules in the modules consumer proguard files to allow the rules to available to both our TV and Mobile apps.

```
-keep class com.google.ads.interactivemedia.** { *; }
-keep interface com.google.ads.interactivemedia.** { *; }
-keep class com.google.obf.** { *; }
-keep interface com.google.obf.** { *; }

```

P.S - The above rules are just an example and might not be the best for your use case.

**Note** - You don't need to name the file as `consumer-rules.pro`. You can name it anything. We only need to make sure that we point to the correct file under the `build.gradle` `consumerProguardFiles` attribute.

## TLDR

Consumer proguard rules have been actively used by third-party libraries and that is how they bundle proguard rules in their AAR. In this new bold world of modularity, every module included in your app can be treated as a third party library and hence can/should house it's own Proguard/R8 rules.  