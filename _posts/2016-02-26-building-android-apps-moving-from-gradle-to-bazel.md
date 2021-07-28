---
layout: post
title: 'Building Android Apps: Moving from Gradle to Bazel'
description: "Bazel is a new build system developed by Google which allows for an alternative to Gradle for building Android apps. I'll show you how you can use Bazel to build your existing Gradle-based Android app."
---

Originally posted
[on Wattpad](https://www.wattpad.com/story/64129482-building-android-apps-moving-from-gradle-to-bazel).

## Why Bazel?

### Intro

Bazel is still in its early days as a build system for Android, but support for building Android apps exists and Google uses Bazel (previously Blaze) for building many of its apps. Google claims Bazel is a lot faster for incremental builds for large apps, which is a good enough reason for me to give it a try, considering how slow Gradle can be.

If you want to try out Bazel for building your app, you should know that there's no need to give up on Gradle. It's possible to use both Gradle and Bazel to build the same project. The rest of this will be a guide for how you can create Bazel build files similar to your build.gradle files so that you can build your project with both Gradle and Bazel. You may want to do this so that you can gradually switch over to using Bazel, or so you can always have something to fall back to in case something goes wrong with your Bazel build. It's also likely that if you have a complex enough project, you'll be using features of Gradle which aren't built in to Bazel, but it may prove to be a reasonable alternative to use during development.

To follow along with this guide, you'll need to get Bazel set up on your machine. I recommend following the instructions at [bazel.io/docs/install](https://bazel.io/docs/install). After setting up, go through the tutorial on how to build an Android app. The tutorial only covers how to make a basic app, with no reference to Gradle, but if you're able to build it, that should mean that you have everything you need to follow this guide.

## build.gradle ➡ WORKSPACE + BUILD

Now we'll start converting over your build.gradle into Bazel's WORKSPACE and BUILD files. This guide will assume that you're using Bazel 0.2.0, which, given the rate Bazel moves at, will most likely be outdated by the time you read this.

### Build directory setup

The first thing you may have to do if you want to be able to build with both Gradle and Bazel is change the build output directory name that Gradle uses. You only need to do this if you're working on a machine with a case-insensitive file system (Windows / Mac OS). This is because Gradle uses "build" as the name for the directory and Bazel uses BUILD as the name for its build files.

This can be done by setting the buildDir property in your project level build.gradle as seen below.

```gradle
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.5.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

// Change the build output folder of Gradle to be something different than BUILD
// so case-insensitive filesystems (Windows and MacOSX) will allow Bazel's BUILD
buildDir = 'gen_build'
allprojects {
    buildDir 'gen_build'
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

### Basic Bazel setup for a Gradle project

You'll need to start by creating two files, WORKSPACE and BUILD. If you went through the Bazel tutorial for building an Android app, these should seem familiar.

The WORKSPACE file should go in the root directory of your project and should look something like the following:

```python
android_sdk_repository(
    name = "androidsdk",
    path = "/path/to/Android/sdk",
    api_level = 23,
    build_tools_version = "23.0.2"
)
```

build_tools_version corresponds to buildToolsVersion in build.gradle, api_level corresponds to compileSdkVersion and path refers to your SDK directory, which should be the same as the sdk.dir property in local.properties.

The BUILD file should be created in the app directory if you're using the standard directory layout for Android Gradle projects.

```python
android_binary(
    name = "YourApp",
    custom_package = "com.example.yourapp",
    manifest = "src/main/AndroidManifest.xml",
    srcs = glob(["src/main/java/**"]),
    resource_files = glob(["src/main/res/**"]),
)
```

name is a property that will be used when building / running your project. custom_package should correspond to applicationId in your build.gradle. manifest, srcs and resource_files correspond to your AndroidManifest.xml, Java source files and resource files respectively. Note that the paths specified here are relative to the directory that the BUILD file is in.

### Building with support library dependencies

The last thing you'll need in order to build an newly created Android project from Android Studio is the support library dependencies.

This can be done by including the libraries as deps in your BUILD file.

```python
android_binary(
    name = "YourApp",
    custom_package = "com.example.yourapp",
    manifest = "src/main/AndroidManifest.xml",
    srcs = glob(["src/main/java/**"]),
    resource_files = glob(["src/main/res/**"]),
    deps = [
        "@androidsdk//:appcompat_v4",
        "@androidsdk//:appcompat_v7",
        "@androidsdk//:design"
    ],
)
```

The naming may seem odd, but appcompat_v4 refers to support_v4.

Note that unlike Gradle, you can't exclude the appcompat_v4 dependency and expect it to be included because of appcompat_v7.

At this point it is now possible to build a new Android Studio project with Bazel, but you probably want to be able to do more than just that. Continue reading to find out how to include remote dependencies in your project.

## Remote Dependencies

### Including a remote jar from a maven repository

First we'll start with including RxJava from a remote maven repository. All that's needed is to make the following changes to your WORKSPACE and BUILD files:

```python
android_sdk_repository(
    name = "androidsdk",
    path = "/path/to/Android/sdk",
    api_level = 23,
    build_tools_version = "23.0.2"
)

maven_jar(name = "rxjava", artifact = "io.reactivex.rxjava:1.1.1")
```

```python
android_binary(
    name = "YourApp",
    custom_package = "com.example.yourapp",
    manifest = "src/main/AndroidManifest.xml",
    srcs = glob(["src/main/java/**"]),
    resource_files = glob(["src/main/res/**"]),
    deps = [
        "@androidsdk//:appcompat_v4",
        "@androidsdk//:appcompat_v7",
        "@androidsdk//:design",
        "@rxjava//jar:jar",
    ],
)
```

The name specified in the maven_jar function should match what is specified in deps, but otherwise can be named however you like. The artifact should be the same as how your dependency is specified in build.gradle, i.e. groupId:artifactId:version.

### Including a remote aar from a maven repository

Unfortunately, there is no built-in support for including remote aars with Bazel as of now. If you look at this issue on Github however, there is a workaround [github.com/bazelbuild/bazel/issues/561](https://github.com/bazelbuild/bazel/issues/561).

As an example of an aar that you can include in your app, I'll show Timber. There are three steps required in order to include an aar: updating the WORKSPACE, creating a new BUILD file and including the dependency in your app's BUILD.

For your WORKSPACE, you need to add a new_http_archive rule as follows:

```python
android_sdk_repository(
    name = "androidsdk",
    path = "/path/to/Android/sdk",
    api_level = 23,
    build_tools_version = "23.0.2"
)

# THIS IS A WORKAROUND TO USE AAR LIBRARY
# See https://github.com/bazelbuild/bazel/issues/561 for more detail
new_http_archive(
    name = "timber",
    url = "https://repo1.maven.org/maven2/com/jakewharton/timber/timber/4.1.1/timber-4.1.1.aar",
    type = "zip",
    build_file = "timber.BUILD"
)

maven_jar(name = "rxjava", artifact = "io.reactivex.rxjava:1.1.1")
```

The name is like name in maven_jar and the url points to the remote maven repository which hosts the aar. build_file points to the BUILD file which we'll make now for the aar in the same directory as WORKSPACE.

In the timber.BUILD file, include the following:

```python
android_library(
    name = "library",
    custom_package = "com.jakewharton.timber",
    manifest = "AndroidManifest.xml",
    resource_files = glob(["res/**"]),
    visibility = ["//visibility:public"],
    deps = ["jar"],
)

java_import(
    name = "jar",
    jars = ["classes.jar"],
)
```

Most of this file handles building the aar like how an Android app would be built. What happens is that the aar is extracted as part of the build process and then the manifest, resources and classes are packaged up during the build.

Finally, we'll add the dependency to the BUILD files deps:

```python
android_binary(
    name = "YourApp",
    custom_package = "com.example.yourapp",
    manifest = "src/main/AndroidManifest.xml",
    srcs = glob(["src/main/java/**"]),
    resource_files = glob(["src/main/res/**"]),
    deps = [
        "@androidsdk//:appcompat_v4",
        "@androidsdk//:appcompat_v7",
        "@androidsdk//:design",
        "@timber//:library",
    ],
)
```

## Local Dependencies

Sometimes some SDKs are behind with the times and don't provide maven dependencies. In this case, they may provide a jar, aar or Android Library project which you import into your project to build with.

### Depending on a local jar

As an example of including a local jar, and to demonstrate the common use of the useLibrary property with Gradle, I'll show how to include the Apache HTTP library.

If your project, or one of your dependencies requires use of the legacy Apache HTTP library and you target API level 23+, you need to include the jar manually. It is located at SDK_DIR/platforms/android-API_LEVEL/optional/org.apache.http.legacy.jar.

Copy the jar to somewhere in your project where Gradle won't see it (if you're using useLibrary).

Now all that is needed is to modify your BUILD file to the following:

```python
android_binary(
    name = "YourApp",
    custom_package = "com.example.yourapp",
    manifest = "src/main/AndroidManifest.xml",
    srcs = glob(["src/main/java/**"]),
    resource_files = glob(["src/main/res/**"]),
    deps = [
        "@androidsdk//:appcompat_v4",
        "@androidsdk//:appcompat_v7",
        "@androidsdk//:design",
        "apache-http",
    ],
)

java_import(
    name = "apache-http",
    jars = ["libs/org.apache.http.legacy.jar"]
)
```

The path specified in jars is a relative path from the directory the BUILD file is in.

Note that this is a workaround needed only for Bazel 0.2.0. Future versions of Bazel should allow including this dependency in a similar way to including the support libraries.

### Depending on a local aar

As with a remote aar, a workaround is needed to build with a local aar. The difference this time is that you have to extract the aar yourself before using it.

For this example, I'll show how to build using a local aar of RxAndroid. The steps are similar to using a remote aar. You can first extract the aar to a directory in the root of your project. Then make the following changes to WORKSPACE and your BUILD file, and also add an rxandroid.BUILD to the root of your project.

```python
android_sdk_repository(
    name = "androidsdk",
    path = "/path/to/Android/sdk",
    api_level = 23,
    build_tools_version = "23.0.2"
)

new_local_repository(
    name = "rxandroid",
    path = "rxandroid",
    build_file = "rxandroid.BUILD"
)

maven_jar(name = "rxjava", artifact = "io.reactivex.rxjava:1.1.1")
```

```python
android_binary(
    name = "YourApp",
    custom_package = "com.example.yourapp",
    manifest = "src/main/AndroidManifest.xml",
    srcs = glob(["src/main/java/**"]),
    resource_files = glob(["src/main/res/**"]),
    deps = [
        "@androidsdk//:appcompat_v4",
        "@androidsdk//:appcompat_v7",
        "@androidsdk//:design",
        "@rxjava//jar:jar",
        "@rxandroid//:library",
    ],
)
```

```python
android_library(
    name = "library",
    custom_package = "io.reactivex.rxandroid",
    manifest = "AndroidManifest.xml",
    resource_files = glob(["res/**"]),
    visibility = ["//visibility:public"],
    deps = ["jar"],
)

java_import(
    name = "jar",
    jars = ["classes.jar"],
)
```
