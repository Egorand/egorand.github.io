---
layout: post
title:  "All the things Compat"
date:   2016-01-25 00:00:00 -0400
tags: [android, java]
---
Android is constantly evolving, every year we get new cool features and new awesome APIs added to 
the SDK. But backwards compatibility has always been an issue. Here's the Android versions 
distribution at the time of writing:

![oGaZBEQ](/assets/oGaZBEQ.png)

We're in 2016 already, but KitKat, introduced in September 2013, is still the most popular system 
version with a whopping 36.1%. Two most recent versions, Lollipop and Marshmallow, are represented 
by a total of just above 30% of devices. It means that as a developer, you should at
least support users with Jelly Bean if you want to target 90% of the market. So what does it mean in
terms of APIs you can use?

### New APIs, deprecated APIs

Every new release comes with a set of new APIs, but there's usually a bunch of old APIs that are 
being deprecated. Once in a while you're trying to use a method and notice that Android Studio has 
crossed it out - it's deprecated. Here's a good example:

```java
Resources res = context.getResources();
int colorWhite = res.getColor(android.R.color.white);
```

This surely looks familiar, but guess what? `getColor(int)` has been deprecated in SDK Level 23 and 
replaced with `getColor(int, Theme)`. All right, let's play dumb. The documentation says that the 
`Theme` argument may be `null`, so let's try it out:

```java
Resources res = context.getResources();
int colorWhite = res.getColor(android.R.color.white, null);
```

And looks like we've solved the problem just to create another one: the method we're trying to use 
is only available since SDK Level 23 and will not work on older system versions. What do we do now? 
There is a solution to check which version of Android we're running against by comparing the 
[`Build.VERSIONS.SDK_INT`][sdk-int] value to a specific version code in 
[`Build.VERSION_CODES`][version-codes], calling the new method if the version is high enough, and 
falling back to the deprecated method otherwise. In our case the code will look like this:

```java
Resources res = getResources();
int colorWhite;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    colorWhite = res.getColor(android.R.color.white, null);
} else {
    colorWhite = res.getColor(android.R.color.white);
}
```

We've done our best, but the ugly strikethrough is still there... Is there a better way? Sure there 
is! Luckily, the Android team has provided a bunch of classes, usually postfixed with "Compat", that
are safe to call on most Android versions, and that hide away all the backwards compatibility 
mechanics.

### The Compat classes

Below you'll find a quick survey of the most interesting, and probably the most helpful Compat 
classes that can be found in the compatibility library.

### ContextCompat

The best way to solve the problem outlined in the example above is to use the 
[`ContextCompat`][context-compat] class:

```java
int colorWhite = ContextCompat.getColor(this, android.R.color.white);
```

We don't have to worry about system versions checks, it's all under the hood. A couple more useful 
methods in `ContextCompat` are:

- `final static Drawable getDrawable(Context context, int id)`

Same story as with `getColor()`, the old method in `Resources` has been deprecated and replaced with
a new one that requires a `Theme` argument. Just use the `ContextCompat` version and forget the 
trouble.

- `static int checkSelfPermission(Context context, String permission)`

There were no runtime permissions before Android Marshmallow, so when
called on an older system version, this method will return `PackageManager.PERMISSION_GRANTED` for 
any permission declared in Manifest.

### ResourcesCompat

This is an almost complete list of methods inside the [`ResourcesCompat`][resources-compat] class:

- `int getColor(Resources res, int id, Resources.Theme theme)`
- `static Drawable getDrawable(Resources res, int id, Resources.Theme theme)`
- `static Drawable getDrawableForDensity(Resources res, int id, int density, Resources.Theme theme)`

Those look quite similar to the methods in `ContextCompat`, but allow you to provide the `Theme` 
argument yourself. I've never stumbled upon a use-case for any of these methods, so can't really say
how helpful they are. Another odd detail is that `getDrawable()` and `getDrawableForDensity()` are 
static methods, while `getColor()` is an instance method.

### ActivityCompat

A couple of methods from the [`ActivityCompat`][activity-compat] are:

- `static void requestPermissions(Activity activity, String[] permissions, int requestCode)`

We've already seen the `checkSelfPermission()` from `ContextCompat`, and this method is another part
of the runtime permissions compatibility implementation. On Android versions prior to Marshmallow it
will automatically grant the permission if it has been declared in the Manifest.

- `static void startActivity(Activity activity, Intent intent, Bundle options)`

The flavor of `startActivity()` with the `options` parameter has been added in API 16, and `options`
is typically a `Bundle`, returned by one of the methods in [`ActivityOptions`][activity-options] 
class. `ActivityOptions` has a lot of cool methods to create custom `Activity` launch animations. 
The backwards compatible companion of `ActivityOptions` is 
[`ActivityOptionsCompat`][activity-options-compat]. Here's how it could work:

```java
private void startDetailActivity(ImageView icon, Intent detailActivityIntent) {
    ActivityOptionsCompat options = ActivityOptionsCompat
            .makeScaleUpAnimation(icon, 0, 0, icon.getWidth(), icon.getHeight());
    ActivityCompat.startActivity(this, detailActivityIntent, options.toBundle());
}
```

Unfortunately, the animations themselves haven't been ported to pre-JB, so those methods will have 
no effect on older system versions.

### DrawableCompat

One pretty useful method in [`DrawableCompat`][drawable-compat] is the following:

- `static void setTint(Drawable drawable, int tint)`

This will result in `setTint()` being called on `Drawable` on Lollipop and higher, and will fall 
back to `setColorFilter()` on older versions.

### ViewCompat

[`ViewCompat`][view-compat] has a lot of wrappers to `View` class methods, introduced in Android 
Honeycomb and later, such as `getX()`, `setX()`, `getY()`, `setY()`, `getAlpha()`, `setAlpha()` and 
many more. Be careful though, most of the setters are just no-ops, and most getters return default 
values.

### MenuItemCompat

The `MenuItem` class has received a bunch of new methods since the Action Bar was introduced back in
Honeycomb, and [`MenuItemCompat`][menu-item-compat] makes them fully backwards compatible, so you 
can achieve identical behavior on pre-Honeycomb versions as well. The most helpful methods in my 
opinion are the ones that help you deal with action views and action providers:

- `static View getActionView(MenuItem item)`
- `static boolean expandActionView(MenuItem item)`
- `static boolean collapseActionView(MenuItem item)`
- `static ActionProvider getActionProvider(MenuItem item)`

It's worth mentioning that there are two classes called `ActionProvider`, the 
`android.view.ActionProvider` and the `android.support.v4.view.ActionProvider`. Make sure you use 
the latter with `MenuItemCompat`.

### NotificationCompat

[`NotificationCompat`][notification-compat] contains quite a handful of inner classes that help you 
create highly customized notifications. First use `NotificationCompat.Builder` to provide basic 
content for a notification, then initialize one of the implementations of 
`NotificationCompat.Style`, such as `NotificationCompat.BigPictureStyle` or 
`NotificationCompat.BigTextStyle` and pass it to `NotificationCompat.Builder` via `setStyle()`. 
There's an [Android Training lesson][notifications-training] that describes the process of creating 
custom notifications in more detail.

### AsyncTaskCompat

Finally, [`AsyncTaskCompat`][async-task-compat] has the following interesting method:

- `static <Params, Progress, Result> AsyncTask<Params,
  Progress, Result> executeParallel(AsyncTask<Params, Progress,
  Result> task, Params... params)`

As you probably know, the default behavior of `AsyncTask` changed twice throughout the history of 
Android: it used serial execution (all tasks inside the application were executed on a single worker
thread) in the beginning, then starting with Donut it switched to parallel execution, only to jump 
back to serial on Honeycomb. If you'd like to run your tasks in parallel on newer Android versions, 
use `executeParallel()`, or just call `executeOnExecutor()` instead of `execute()` directly on 
`AsyncTask`, passing in `AsyncTask.THREAD_POOL_EXECUTOR` as the parameter.

### Conclusion

This was just a small selection from the complete collection of Compat classes, available in Android
compatibility libraries, but those are probably the most used ones. But the rule of thumb is, when 
you stumble upon a deprecated method in the Android SDK - first dive into the docs for an 
explanation, and then look for relevant classes with Compat postfix! This is a pretty good way to 
stay on the safe side when dealing with backwards compatibility on Android. Cheers!

[sdk-int]: https://developer.android.com/reference/android/os/Build.VERSION.html#SDK_INT
[version-codes]: https://developer.android.com/reference/android/os/Build.VERSION_CODES.html
[context-compat]: https://developer.android.com/reference/android/support/v4/content/ContextCompat.html
[resources-compat]: https://developer.android.com/reference/android/support/v4/content/res/ResourcesCompat.html
[activity-compat]: https://developer.android.com/reference/android/support/v4/app/ActivityCompat.html
[activity-options]: https://developer.android.com/reference/android/app/ActivityOptions.html
[activity-options-compat]: https://developer.android.com/reference/android/support/v4/app/ActivityOptionsCompat.html
[drawable-compat]: https://developer.android.com/reference/android/support/v4/graphics/drawable/DrawableCompat.html
[view-compat]: https://developer.android.com/reference/android/support/v4/view/ViewCompat.html
[menu-item-compat]: https://developer.android.com/reference/android/support/v4/view/MenuItemCompat.html
[notification-compat]: https://developer.android.com/reference/android/support/v4/app/NotificationCompat.html
[notifications-training]: https://developer.android.com/training/notify-user/index.html
[async-task-compat]: https://developer.android.com/reference/android/support/v4/os/AsyncTaskCompat.html
