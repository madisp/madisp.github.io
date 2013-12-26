---
layout:     post
title:      Stupid - a stupid scripting language for Java/JVM
date:       2013-12-26 15:48:00
categories: android, bad, stupid
---

Some time ago I tried to replicate AngularJS's variant of two-way databinding - which I find
absolutely lovely - in Android. Angular has a simple expression language that one can use within
the templates, e.g., say that one wants to display a div only if user is logged in and is an admin, e.g.:

```xml
<div ng-show="user.logged_in && user.is_admin">
	<button ng-click="doScaryStuff()" />
</div>
```

The expression language in Angular is fairly basic but can quickly lead to very powerful results. Lets solve the same problem in Android, i.e. we want to hide a button. One usually has to write some Java code that typically ends up in an activity or fragment.

Some layout:

```xml
<Button android:id="@+id/scaryButton"
	android:layout_width="wrap_content"
	android:layout_height="wrap_content"
	android:text="So scary"
	android:click="doScaryStuff()"
	android:visibility="gone" />
```

And matching code:

```java
public void onCreate(Bundle savedInstanceState) {
	// lets assume we have a user object
	if (user.isLoggedIn() && user.isAdmin()) {
		findViewById(R.id.scaryButton).setVisibility(View.VISIBLE);
	}
}
```

This kind-of works, but we are already opening a can of worms with state-handling. What happens if the device configuration changes (state save-restore event)? What if our model for the user instance changes?

Usually to have the layout represent some business logic state one has to set up onSaveInstanceState/onRestoreInstanceState, a bunch of broadcastreceivers/callbacks to receive notifications about changes and the actual viewtree manipulation code on top of that. Needless to say, I really like AngularJS's approach.

So it would be really-really nice if we could do something similar to the AngularJS example in Android. To achieve this I needed a scripting language that I could use in XML layouts and a lightweight interpreter that for it that I could run in Java.

Enter Stupid.