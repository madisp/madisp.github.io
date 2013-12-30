---
layout:     post
title:      Android Drawable Cache Collisions, Part 1
date:       2013-12-30 01:00:25
categories: android
---

```java
boolean isColorDrawable = false;
if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT &&
		value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
	isColorDrawable = true;
}
Drawable dr = getCachedDrawable(isColorDrawable ? mColorDrawableCache : mDrawableCache, key);
```

Note! This post contains [source code](http://androidxref.com/4.0.3_r1/xref/frameworks/base/core/java/android/content/res/Resources.java) from the Android Open Source Project, licensed under the Apache License 2.0.

***HELP! My Drawables Aren't Loading on Gingerbread Devices***

Pre-Honeycomb Android versions have a really weird bug where sometimes loading a drawable resource loads a transparent color drawable instead.

Some cases of this in the wild:

* [Strange R.java issue cause drawable resource isn't load correctly (StackOverflow)](http://stackoverflow.com/questions/12677889/strange-r-java-issue-cause-drawable-resource-isnt-load-correctly)
* [Weird problem about an android drawable (StackOverflow)](http://stackoverflow.com/questions/4435099/weird-problem-about-an-android-drawable)
* [ImageButton cannot use first alphabetical drawable in the res/drawable folder (AOSP bug tracker)](https://code.google.com/p/android/issues/detail?id=20283)

The fix, if you need to support Gingerbread or older, is to usually add a *aaaa.png* or similarly named file to your project. If you have a lot of drawables then sometimes even that approach can fail, I've seen it happen. Because the fix is so weird I *really* needed to find out what's going on.

***TL;DR*** Before we move on, a quick fix that works even when the *aaaa.png* trick fails. Replace your `0x00000000` (transparent black) color resources with `0x00ffffff` (transparent white).

***Quick Investigation into the Patch***

First off, lets take a look at the [patch](https://android-review.googlesource.com/#/c/15815/4/core/java/android/content/res/Resources.java) introduced in Honeycomb. The commit adds a new `LongSparseArray` called `mColorDrawableCache`. Both colors and drawables used to share the same cache - `mDrawableCache` - but apparently the fix was to split them up. This is really interesting and basically points to a hash function / hashmap key collision between colors and drawables, i.e., collisions between a single type of drawables should still be non-existent.

***Digging into the Caching Key***

Lets take a look at how the hash is calculated in *Resources#loadDrawable*:

```java
/*package*/ Drawable loadDrawable(TypedValue value, int id)
	throws NotFoundException {

	if (TRACE_FOR_PRELOAD) {
		// Log only framework resources
		if ((id >>> 24) == 0x1) {
			final String name = getResourceName(id);
			if (name != null) android.util.Log.d("PreloadDrawable", name);
		}
	}

	final long key = (((long) value.assetCookie) << 32) | value.data;
	boolean isColorDrawable = false;
	if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT &&
			value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
		isColorDrawable = true;
	}
	Drawable dr = getCachedDrawable(isColorDrawable ? mColorDrawableCache : mDrawableCache, key);

	if (dr != null) {
		return dr;
	}
	// loadDrawable is really long, but the rest is not as interesting -madis
	...
}
```

The key is defined on this line:

```java
final long key = (((long) value.assetCookie) << 32) | value.data;
```

The [documentation](https://developer.android.com/reference/android/util/TypedValue.html#assetCookie) gives us the following insights into `assetCookie` and `data`:

> public int **assetCookie**<br />
> Additional information about where the value came from; only set for strings.
>
> public int **data**<br />
> Basic data in the value, interpreted according to *type*

So the caching key is a 64-bit signed integer where the first part, 32-bits, is *information about where the value came from* and the second part, also 32-bits, is the actual data of the value. The data field, as the docs say, is interpreted according to the type. For non-color drawables the type is `0x03` - a string.
Indeed, calling `getString(R.drawable.ic_launcher)` returns `res/drawable/ic_launcher.png` which is the path to the bitmap. Checking the native source of [ResourceTypes.h](http://androidxref.com/4.0.3_r1/xref/frameworks/base/include/utils/ResourceTypes.h#232) gives some insights into how the data field is interpreted (parts omitted):

```cpp
// Type of the data value.
enum {
	// ...
	// The 'data' holds an index into the containing resource table's
	// global value string pool.
	TYPE_STRING = 0x03,
	// ...
	// The 'data' is a raw integer value of the form #aarrggbb.
	TYPE_INT_COLOR_ARGB8 = 0x1c,
	// The 'data' is a raw integer value of the form #rrggbb.
	TYPE_INT_COLOR_RGB8 = 0x1d,
	// The 'data' is a raw integer value of the form #argb.
	TYPE_INT_COLOR_ARGB4 = 0x1e,
	// The 'data' is a raw integer value of the form #rgb.
	TYPE_INT_COLOR_RGB4 = 0x1f,
	// ...
};
```

So, we have a problem if a drawable's index to the resource table's string pool happens to have the same value as a color. A likely happenstance if you have a lot of drawables and a low-value color, say, `0x00000000`. This also means that we should be able to reproduce with a super-simple app having only a single bitmap and a transparent color. Let's try.

***Proof of Concept Reproduction***

I [created a small app](https://github.com/madisp/android-drawable-collision) that exhibits the problem. Source is here, starting with *src/main/AndroidManifest.xml*:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.madisp.android.collision"
    android:versionCode="1"
    android:versionName="1.0" >
    <uses-sdk
        android:minSdkVersion="7"
        android:targetSdkVersion="19" />
    <application
        android:allowBackup="true"
        android:icon="@drawable/ic_launcher"
        android:label="Android Drawable Collision" >
        <activity
            android:name=".MainActivity" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

.. and *src/main/res/values/colors.xml*:


```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="transparent">#00000000</color>
</resources>
```

.. and finally *src/main/java/com/madisp/android/collision/MainActivity.java*:

```java
package com.madisp.android.collision;

import android.app.Activity;
import android.os.Bundle;
import android.view.ViewGroup;
import android.widget.ImageView;

public class MainActivity extends Activity {
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		ImageView iv = new ImageView(this);
		iv.setBackgroundResource(R.color.transparent);
		iv.setImageResource(R.drawable.ic_launcher);
		final int matchParent = ViewGroup.LayoutParams.MATCH_PARENT;
		setContentView(iv, new ViewGroup.LayoutParams(matchParent, matchParent));
	}
}
```

If we run the apk we don't get a blurry android. Lets dump the resource table with aapt and we can validate that, yep, we have reproduced the collision:

```
$ aapt dump --values resources android-drawable-collision-debug-unaligned.apk
Package Groups (1)
Package Group 0 id=127 packageCount=1 name=com.madisp.android.collision
  Package 0 id=127 name=com.madisp.android.collision typeCount=3
    type 0 configCount=0 entryCount=0
    type 1 configCount=1 entryCount=1
      spec resource 0x7f020000 com.madisp.android.collision:drawable/ic_launcher: flags=0x00000000
      config (default):
        resource 0x7f020000 com.madisp.android.collision:drawable/ic_launcher: t=0x03 d=0x00000000 (s=0x0008 r=0x00)
          (string16) "res/drawable/ic_launcher.png"
    type 2 configCount=1 entryCount=1
      spec resource 0x7f030000 com.madisp.android.collision:color/transparent: flags=0x00000000
      config (default):
        resource 0x7f030000 com.madisp.android.collision:color/transparent: t=0x1c d=0x00000000 (s=0x0008 r=0x00)
          (color) #00000000
```

Notice the `d=0x00000000` for both our drawable and the transparent color. If we rewrite *colors.xml* so that transparent is `0x00ffffff` we don't get the collision anymore.

In part 2 we'll take a look at how Android generates the resource table and what is *an index into the containing resource table's global value string pool*.