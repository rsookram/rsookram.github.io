---
layout: post
title: Reuse of ViewModel-Backed Fragments
blog: Building at Wattpad
external_link: https://medium.com/tech-wattpad/reuse-of-viewmodel-backed-fragments-7497147c50de
---

At Wattpad, we recently started using Android Architecture Components as the
preferred architecture in our core Android app. One of the benefits of this
approach is that it is easy to test logic in ViewModels. This led us to
extracting as much logic as possible from our Activities into ViewModels so
that we could cover it by tests.

This post describes a solution to a problem we ran into with communication
between Fragments and Activities, which involved logic in our Activities that
we wanted to extract. It happened specifically in the case of accessing an
Activity’s ViewModel from a Fragment which was used in multiple Activities,
each with their own ViewModel.


## Motivation

Fragments often need to communicate with their host (Activity) since they are
only one part of a screen, and changes which happen within a Fragment may need
to affect other parts of the UI in the same screen.

Another aspect of Fragments being only one part of a screen is that they can be
reused on different screens. The problem that this causes is that a direct
reference to a specific Activity can’t be used if the Fragment needs to be
reused in different Activities. This was a problem for us as the core Wattpad
app contains approximately 90 Activities, making it difficult to quickly move
towards a single-Activity architecture.

One recent case where we ran into this problem was when revamping how muting
and unmuting users works. When muting or unmuting, we show a confirmation
dialog which then results in the mute / unmute action happening. This dialog is
implemented as a DialogFragment, and can be shown in different Activities where
you can mute a user from: on a user’s profile, in your inbox, and from within a
thread of private messages.


## Existing Solutions

Our previous solution to this problem was similar to what is described in
[this StackOverflow post](https://stackoverflow.com/questions/14247954/communicating-between-a-fragment-and-an-activity-best-practices/25392549#25392549).
This involves having each Activity which hosts a reusable Fragment implement a
callback interface defined by the Fragment. The Fragment then casts its host
Activity to that interface in its `onAttach` callback and nulls the reference
in `onDetach` like the following:

```kotlin
interface Listener {
    fun onChanged()
}

private var listener: Listener? = null

override fun onAttach(activity: Activity) {
    super.onAttach(activity)
    listener = activity as Listener
}

override fun onDetach() {
    super.onDetach()
    listener = null
}
```

One alternative to this would be to create a separate Fragment for each Activity
that needs to use it, but that can get out of hand when there are many
Activities which need to use the Fragment.


## What does Google say about this?

Google has started offering more advice on how to architect apps since
releasing Android Architecture Components, so we looked to them next to see
what they suggested to do in this case.

When ViewModels were introduced, Google started recommending them as an
alternate approach, where a ViewModel created within an Activity could be
shared with the hosted Fragment to allow for this communication. This works
without problem in a single-Activity app which Google also suggests now, but in
our case we couldn't use this directly given that we have a multi-Activity app,
where Fragments are included in different Activities.

Following Google's recommendation while allowing for the Fragment to be reused
could be done with the StackOverflow solution mentioned above, but requires
additional forwarding of events from the Activity to the ViewModel. This would
be done by having the Activity implement the listener interface which the
Fragments would call, and the implementation of that method could call a method
on the ViewModel.

What we'd like to do ideally is have the Fragment refer to the ViewModel
directly, without needing to go through the Activity. This results in a problem
similar to before where we don't want to refer to a specific ViewModel, since
each Activity may have its own ViewModel.

We can try to use a solution similar to what we had originally, which would be
to have our ViewModels implement an interface defined by the Fragment:

```kotlin
class MuteUserDialogFragment : DialogFragment() {
    interface Listener {
        fun onChanged()
    }
}

class ProfileViewModel : ViewModel(), MuteUserDialogFragment.Listener {
    override fun onChanged() {
    }
}
```

But when we try to do this, we end up with a problem when trying to get the
ViewModel within the Fragment: which class do we pass for the ViewModel?


```kotlin
val profileViewModel by activityViewModels</* Which type? */>()
```

If we pass ProfileViewModel, our Fragment will be tied to the Activity which
hosts ProfileViewModel. This differs from our previous solution in that we
don't have a ViewModel passed into the Fragment like how the Activity was
passed in onAttach.


## Our Solution

The solution we went with for this problem was to provide the Class object to
the Fragment when it is created. This was done through the `arguments` provided
to the Fragment. Only types which can be put into a Bundle can be included in
`arguments`, but luckily for us, `Serializable` is one of those types, and
`Class` implements it.

People have argued that `Serializable` is slow on Android relative to
`Parcelable`, and to always prefer `Parcelable` for this reason. In this case,
the serialization is happening on the main thread as well, but it happens
infrequently. We haven't seen any noticeable slowdown from using this approach
so far, so we've stuck with using `Serializable` for simplicity.

Another thing that can be done when providing the Class argument to the
Fragment is using a
[`where` clause](https://kotlinlang.org/docs/tutorials/kotlin-for-py/generics.html#constraints)
to specify the type constraint. This allows the type to specify that it must be
both a `ViewModel` subclass, and implement the given interface. It looks
similar to the following for the mute dialog case mentioned earlier:

```kotlin
fun <T> newInstance(
    username: String,
    viewModelClass: Class<T>
): DialogFragment where T : Listener, T : ViewModel =
    MuteUserDialogFragment().apply {
        arguments = bundleOf(
            ARG_VIEW_MODEL_CLASS to viewModelClass,
            ARG_USERNAME to username
        )
    }
```

This approach can work with either using ViewModelProviders directly, or if
you're using fragment-ktx, with createViewModelLazy.


## Conclusion

By requiring the ViewModel to only implement an interface, the Fragment can be
used from multiple Activities, each with their own ViewModel, as long as the
ViewModels implement said interface. This allows the ViewModel to handle any
business logic for the Fragments, without introducing more code to wire things
together in the Activity.
