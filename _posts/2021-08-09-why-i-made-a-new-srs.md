---
layout: post
title: Why I Made a New SRS
description: Lately I've been working on a new Spaced Repetition Software (SRS) app for Android, and I figured that I'd write about why I've been doing it, and how it works.
---

Lately I've been working on a [new Spaced Repetition Software (SRS) app](https://github.com/rsookram/srs) for Android,
and I figured that I'd write about why I've been doing it,
and how it works.

## What I use SRS for

I'm not going to cover [what SRS is](https://en.wikipedia.org/wiki/Spaced_repetition#Software) or [why it's a good idea](https://refold.la/roadmap/stage-0/c/active-study/),
but I will mention a little about my history with SRS.

I first came across SRS in the form of [Anki](https://apps.ankiweb.net/) (on desktop) sometime around 2009 when I was learning Japanese,
and I've been convinced since then that it's a useful tool for retaining information.
I've been using it on and off since then,
and the last time I used it was from 2016 ~ 2020,
when I used a fork of [AnkiDroid](https://github.com/ankidroid/Anki-Android).

Earlier this year,
I started learning Korean,
and I got back into using SRS because of that.
I decided to use AnkiDroid again,
and forked it again to make changes to be able to follow the [low-key Anki](https://refold.la/roadmap/stage-1/a/anki-setup/) approach,
since the built-in settings weren't flexible enough to support it.

## Why not use AnkiDroid?

Although I used this fork for a while,
I didn't want to maintain a fork in the long term since it's a fairly large project,
and it could end up being a lot of work to maintain it by myself.
There were also more things about it that I wanted to change.

When it came to the UX of the app,
the biggest thing I wanted to change was the use of the navigation drawer,
because it doesn't play well with gesture-based navigation.
There were also a few other small things that mainly involved there being too many UI elements that didn't provide useful information to me.

There was something deeper in the codebase that I wanted to change though.
AnkiDroid has permission to use the filesystem directly, without going through the Storage Access Framework (SAF).
Avoiding the SAF results in better UX,
but after seeing the changes Google made to Android over the past few years,
I figured that this wouldn't last forever.
Maintaining a fork would have meant taking on risk that major changes may be needed to support a future version of Android.

The remaining reasons are related to the tech stack,
which I wanted to change to something I preferred working with.
The first is the use of Java.
Even though the [migration to Kotlin has started](https://github.com/ankidroid/Anki-Android/pull/8968),
it's going to take a long time because the project is large.
According to [tokei](https://github.com/XAMPPRocky/tokei),
the latest version ([2.15.6](https://github.com/ankidroid/Anki-Android/releases/tag/v2.15.6)) is > 100K lines of Java.

```shell
$ tokei -t Java
-------------------------------------------------------------------------------
 Language            Files        Lines         Code     Comments       Blanks
-------------------------------------------------------------------------------
 Java                  619       128001        85319        22574        20108
```

And since Java is being used,
that means that traditional `View`s are also used.
Given how much of AnkiDroid is still in Java,
it's likely that the migration to Jetpack Compose will take even longer
(if it even happens).
A related note on `View`s is that `WebView`s are used for displaying cards.
I would occasionally run into problems because of this,
and my needs were simple enough to get away without needing a `WebView`.

## DIY

Thankfully Anki uses a [simple](https://eshapard.github.io/anki/target-an-80-90-percent-success-rate-in-anki.html) spaced repetition algorithm,
and I only used a small subset of its features,
so I decided to build my own.
Around that time,
Compose was getting close to its first stable release,
and having played with it since the first dev release,
I wanted to give it a try on a slightly larger project.
What I ended up with was a small,
relatively simple app,
that does what I need and nothing more.

So that's why I've been working on a new SRS.
When I get some more time,
I'm planning on writing about some of the things I learned while building the app (hopefully soon).
For now,
the app should be mostly complete in terms of features.
I've just started using it though,
so things may change a little if things aren't quite to my liking.
