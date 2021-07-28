---
layout: post
title: 'Slow TextView is Slow'
description: "It's no secret that TextView is slow, but you end up seeing far more slowness when you have lots of text in a single TextView. In the Wattpad writer on Android, people write entire chapters in a single EditText, so we come across performance issues most people never see."
---

![ANR in Wattpad writer](/assets/image/slow-textview-writer-anr.jpg)

Originally posted [on Wattpad](https://www.wattpad.com/245272242-adventures-in-androidland-slow-textview-is-slow).

It's no secret that TextView is slow, but you end up seeing far more slowness when you have lots of text in a single TextView. In the Wattpad writer on Android, people write entire chapters in a single EditText, so we come across performance issues most people never see.

One of the major performance issues we solved recently in the writer had to do with when there were many spans in the text. Users would occasionally report this problem as the writer being so slow that it was unusable, or that the app was freezing on them when they were writing. When looking through the logs from these reports, it was clear that this was happening only when users used lots of markup when writing.

From an Android perspective, the only difference that markup made was that there would be at least one span per piece of markup. This includes things like StyleSpans for bold or italics, UnderlineSpans for underlines and AlignmentSpans for paragraph alignment. We weren't really doing anything special in our code when text was added other than updating the word count, and disabling that didn't help either. We needed to dig deeper to find the cause of the problem.

The first tool we turned to was the CPU and GPU monitors within Android Studio itself. When running it and starting to type in the part, it became clear what the problem was.

[![GPU usage drops to 0 during high sustained CPU usage](/assets/image/slow-textview-profile-before.png)](/assets/image/slow-textview-profile-before.png)

As soon as we started typing, there would be no activity on the GPU monitors, and the CPUs would max out. This suggested that there was too much work being done, and that it was blocking the main thread.

The next tool to turn to was Traceview. We knew exactly under which conditions the problem occurred in, and that the CPU was being hogged, so we had to find the culprit. Running Traceview, we ended up seeing a graph which looked like this:

[![Flame chart of SpannableString#sendSpanChanged being called repeatedly](/assets/image/slow-textview-traceview.png)](/assets/image/slow-textview-traceview.png)

It couldn't be any clearer. SpannableStringBuilder#sendSpanChanged was being called excessively when we started typing. At this point we were wondering two things: one, what does this method do and why is it being called so often (text is simply being inserted, no spans should be changing) and two, it's an Android class, so what would we be able to do about it? Both of these questions can be answered by one source... the source code of the Android SDK.

The implementation of SpannableStringBuilder varies between releases of Android, but on all versions that we tried, this problem existed. This meant that there should be something in common between all of them. We started by looking at the method in question: sendSpanChanged. There wasn't anything special in this method, it was simply notifying all the listeners it had that a span changed. We needed to go further up the call stack.

In this case, sendToSpanWatchers was the one calling sendSpanChanged, and the logic to determine whether a span had changed was found here. There were multiple conditions which were checked to determine whether a span changed, however there were only two that were relevant to our case.

```java
int previousSpanStart = spanStart;
if (spanStart > newReplaceEnd) {
    if (nbNewChars != 0) {
        previousSpanStart -= nbNewChars;
        spanChanged = true;
    }
```

This check (and the corresponding one for the end of the span) was checking whether the span start or end changed. spanStart and spanEnd are for the offset of the span within the string, and the writer displays a single string with the text of the entire part. This meant that by inserting a character, the offsets of all the spans after the insertion point would change, and each of them would trigger a notification.

Now that we knew what the root cause was, there were two questions remaining. Why are these span change notifications necessary in the first place and what can we do about this?

The first of the remaining questions was answered by simply looking at the caller of sendToSpanWatchers.

```java
// Span watchers need to be called after text watchers, which may update the layout
sendToSpanWatchers(start, end, newLen - origLen);
```

When a span changes, it's possible that the layout (as in android.text.Layout) may need to be updated to reflect the changes. Layout manages things like the number of lines of text, and their positions and heights. Luckily for us, none of the spans we add need to modify the data it holds. This lead to the idea of preventing these calls altogether from happening.

This is the point where the Android SDK provides us with exactly what we need: TextView#setEditableFactory and TextView#setSpannableFactory. These APIs allow us to return something other than a SpannableStringBuilder for holding the backing text for the EditText.

The easiest thing to do would be to create a subclass of SpannableStringBuilder, but the methods we would need to override were private, so this wasn't a valid approach. We ended up creating a copy of SpannableStringBuilder which had slight modifications to allow us to override the behaviour we wanted to change. The change was to simply not call through to the default implementation of sendSpanChanged when the span which changed was one of the spans we added for markup in the writer. We limited it to these spans since they should be the only ones where it would be possible to have many of, and it allowed the Layout to be updated normally for any spans the system may add.

With the changes applied, this is what the monitors look like:

[![Low CPU and GPU usage with a few short spikes of GPU usage](/assets/image/slow-textview-profile-after.png)](/assets/image/slow-textview-profile-after.png)

There are spikes in GPU activity when typing characters, but the corresponding activity on the CPU is now minimal, and no longer blocks the main thread.

That's what we've done recently for improving performance in the writer. We were quite happy with the solution, since it didn't involve using any private or hidden APIs, and it seemed to work well across different versions of Android. And it worked for our users as well. After releasing this change, we received less reports on slowness in the writer.

If you'd like to see how this all comes together, refer to the following gist. It's mostly comprised of copies of Android classes in order to make the necessary modifications, but also includes our modified SpannableStringBuilder which ignores certain calls to sendSpanChanged. [https://gist.github.com/rashadsookram/d056733dca21c88835143d190dae4fa7](https://gist.github.com/rashadsookram/d056733dca21c88835143d190dae4fa7)
