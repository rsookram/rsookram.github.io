---
layout: post
title: 'Testing Composables with Robolectric'
description: 'Back in Jetpack Compose 1.0.0-beta09, it was announced that Compose tests could be run on Robolectric. With some trial and error, I got it to work, and I’m writing this down for anyone else who wants to do the same, or runs into the same problems I did.'
---

Back in Jetpack Compose [1.0.0-beta09](https://developer.android.com/jetpack/androidx/releases/compose-ui#1.0.0-beta09),
it was announced that Compose tests could be run on [Robolectric](https://github.com/robolectric/robolectric).
After looking through the [official documentation](https://developer.android.com/jetpack/compose/testing) however,
I wasn't able to find an explanation of how to set it up.
With some trial and error,
I got it all to work,
and I'm writing this down for anyone else who wants to do the same,
or runs into the same problems I did.
The code written here works with Compose 1.0.1,
Robolectric 4.6.1,
and AGP 7.0.1.

If you just want to see how to make it work,
and not all the problems you may run into,
skip to [the end](#tldr).

## Getting `ComposeTestRule` to work

To start off,
I added the [dependencies](https://developer.android.com/jetpack/compose/testing#setup) specified in the official documentation to my project.
It didn't make sense to use the `androidTestImplementation` configuration though,
since I wanted the tests to run on a local JVM.
So instead I used `testImplementation` and specified the dependencies like:

```gradle
testImplementation("androidx.compose.ui:ui-test-junit4:$compose_version")
debugImplementation("androidx.compose.ui:ui-test-manifest:$compose_version")
```

After that,
I created a single JUnit 4 test to check the setup:

```kotlin
class ComposeTest {

    @get:Rule val rule = createComposeRule()

    @Test
    fun emptyTest() {
    }
}
```

Without the `Rule`,
`emptyTest` would pass because it does nothing.
But as-is,
the test failed with the following exception:

```
java.lang.ExceptionInInitializerError
	at androidx.test.ext.junit.rules.ActivityScenarioRule.lambda$new$0$ActivityScenarioRule(ActivityScenarioRule.java:70)
	...
	at androidx.compose.ui.test.junit4.android.ComposeRootRegistry$getStatementFor$1.evaluate(ComposeRootRegistry.android.kt:150)
	...
Caused by: java.lang.RuntimeException: Method allowThreadDiskReads in android.os.StrictMode not mocked. See http://g.co/androidstudio/not-mocked for details.
	at android.os.StrictMode.allowThreadDiskReads(StrictMode.java)
	at androidx.test.internal.platform.ServiceLoaderWrapper.loadService(ServiceLoaderWrapper.java:42)
	at androidx.test.internal.util.Checks.<clinit>(Checks.java:132)
	... 52 more
```

The solution to this problem is included in the error message.
I was using Gradle's Kotlin DSL,
so I added `isReturnDefaultValues = true` to my `build.gradle.kts`:

```gradle
android {
    // ...
    testOptions {
        unitTests {
            isReturnDefaultValues = true
        }
    }
}
```

I then re-ran the tests,
and saw another `Exception` thrown:

```
java.lang.IllegalStateException: No instrumentation registered! Must run under a registering instrumentation.
	at androidx.test.platform.app.InstrumentationRegistry.getInstrumentation(InstrumentationRegistry.java:45)
	...
	at androidx.compose.ui.test.junit4.android.ComposeRootRegistry$getStatementFor$1.evaluate(ComposeRootRegistry.android.kt:150)
	...
```

This makes sense as the instrumentation is normally provided at runtime when running on a real Android OS.
I wanted to use Robolectric though,
and didn't specify that yet.
Annotating the test suite with `@RunWith(RobolectricTestRunner::class)` fixes this problem.
But after that,
I ended up running into another problem:

```
No such manifest file: ./AndroidManifest.xml

Unable to resolve activity for Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=org.robolectric.default/androidx.activity.ComponentActivity } -- see https://github.com/robolectric/robolectric/pull/4736 for details
java.lang.RuntimeException: Unable to resolve activity for Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=org.robolectric.default/androidx.activity.ComponentActivity } -- see https://github.com/robolectric/robolectric/pull/4736 for details
	at org.robolectric.android.internal.RoboMonitoringInstrumentation.startActivitySyncInternal(RoboMonitoringInstrumentation.java:84)
	...
```

This error meant that the [manifest from `ui-test-manifest`](https://github.com/androidx/androidx/blob/1258722baedeeb0e7af33b2367bd4bff74982cbe/compose/ui/ui-test-manifest/src/main/AndroidManifest.xml) wasn't being found,
and the issue mentioned in this error message doesn't quite help in this case.
What I needed to do is ensure that my tests [had access to Android resources](https://developer.android.com/training/testing/fundamentals#small-tests) by setting `isIncludeAndroidResources` to true.

With all that,
the empty test finally passed.

## Getting a Compose test to work

Now I went to write a test that actually used Compose by using `ComposeContentTestRule.setContent`:

```kotlin
@Test
fun textIsDisplayed() {
    rule.setContent {
        Text("Testing Compose")
    }

    rule.onNodeWithText("Testing Compose").assertIsDisplayed()
}
```

This time,
I ran into an `IllegalAccessException` when `ModernAsyncTask` was being accessed:

```
java.util.concurrent.ExecutionException: java.lang.RuntimeException: java.lang.IllegalAccessException: class androidx.test.espresso.base.ThreadPoolExecutorExtractor$2 cannot access a member of class androidx.loader.content.ModernAsyncTask with modifiers "public static final"
java.lang.RuntimeException: java.util.concurrent.ExecutionException: java.lang.RuntimeException: java.lang.IllegalAccessException: class androidx.test.espresso.base.ThreadPoolExecutorExtractor$2 cannot access a member of class androidx.loader.content.ModernAsyncTask with modifiers "public static final"
	...
	at androidx.compose.ui.test.junit4.AndroidComposeTestRule.waitForIdle(AndroidComposeTestRule.android.kt:286)
	at androidx.compose.ui.test.junit4.AndroidComposeTestRule.setContent(AndroidComposeTestRule.android.kt:281)
	at io.github.rsookram.example.ComposeTest.emptyTest(ComposeTest.kt:17)
	...
```

Searching for that error in Robolectric's Github repository turned up [this issue](https://github.com/robolectric/robolectric/issues/6593).
The workaround for now is to have Robolectric instrument the `androidx.loader.content` package.
This can be done by setting the `instrumentedPackages` property when [configuring Robolectric](http://robolectric.org/configuring/).

I configured Robolectric using `@Config`,
and with that,
I was finally able to run a Compose test using Robolectric.
The final result looked like this:

```kotlin
@Config(instrumentedPackages = ["androidx.loader.content"])
@RunWith(RobolectricTestRunner::class)
class ComposeTest {

    @get:Rule val rule = createComposeRule()

    @Test
    fun textIsDisplayed() {
        rule.setContent {
            Text("Testing Compose")
        }

        rule.onNodeWithText("Testing Compose").assertIsDisplayed()
    }
}
```

## <a name="tldr" /> TL;DR

The next time I need to set up Robolectric for Compose in a project,
these are the steps that I'll take to make sure everything works:

1. Add the dependencies to the project:
  - `androidx.compose.ui:ui-test-junit4` and Robolectric under the `testImplementation` configuration
  - `androidx.compose.ui:ui-test-manifest` under the `debugImplementation` configuration
1. Configure `testOptions` in the Gradle configuration by setting `isReturnDefaultValues` and `isIncludeAndroidResources` to `true`.
1. Run the test suite using Robolectric by setting the test runner with `@RunWith`.
1. Instrument the `androidx.loader.content` package with `@Config` or
   `robolectric.properties`.

If you want to see how this all comes together,
take a look at [this PR](https://github.com/rsookram/srs/pull/19/files) where I added Compose tests to the SRS app I'm working on.
