---
layout: post
title:  "Taking Photos Not So Simply: How I Got Bitten By ACTION_IMAGE_CAPTURE"
date:   2016-04-05 00:00:00 -0400
categories: android
tags: [android, java]
---
I've worked on a number of apps that are taking pictures for different use cases, and since it has 
always been a secondary feature for those apps, I've relied on using Intents, the approach described 
in [Taking Photos Simply][taking-photos-simply]. Intents are powerful, they can save you lots of 
man-hours by providing the possibility to reuse the functionality of other apps inside the system, 
rather than reinventing the wheel. Additionally, as explained in 
[Permissions Best Practices][permissions], Intents can help you keep the number of permissions that 
your app uses low, making it more trustworthy. However, today I've stumbled upon a very interesting 
edge case related to `MediaStore.ACTION_IMAGE_CAPTURE`, which really made me wonder if everything is
that easy.

I'm working on a library that gets integrated by two apps. Internally, the library takes pictures by
firing an `Intent` with `MediaStore.ACTION_IMAGE_CAPTURE`. Testing discovered that whenever the 
image capturing is invoked in one of the apps, the app crashes with roughly the following error 
message:

```java
ActivityManager: Permission Denial: starting Intent
{ act=android.media.action.IMAGE_CAPTURE flg=0x3000000 pkg=com.google.android.GoogleCamera
cmp=com.google.android.GoogleCamera/com.android.camera.CaptureActivity
(has clip) (has extras) } from null (pid=-1, uid=10098)
with revoked permission android.permission.CAMERA
```

The `with revoked permission android.permission.CAMERA` looked troubling: indeed, the app requires 
`android.permission.CAMERA` for a different feature and hasn't been requested at runtime yet, but 
what does it have to do with the `Intent`? I started googling and found an [issue report][issue], 
describing this exact problem. Interestingly enough, the resolution is "WorkingAsIntended", and 
indeed the [documentation][action-image-capture] for `ACTION_IMAGE_CAPTURE` says:

> Note: if you app targets M and above and declares as using the CAMERA permission which is not 
> granted, then atempting to use this action will result in a SecurityException.

This is how this behavior is explained by a Google engineer, commenting on the ticket:

> This is intended behavior to avoid user frustration where they revoked the camera permission from 
> an app and the app still being able to take photos via the intent. Users are not aware that the 
> photo taken after the permission revocation happens via different mechanism and would question the
> correctness of the permission model. This applies to MediaStore.ACTION_IMAGE_CAPTURE, 
> MediaStore.ACTION_VIDEO_CAPTURE, and Intent.ACTION_CALL the docs for which document the behavior 
> change for apps targeting M.

That was a very interesting finding. The rationale makes sense to me, however, I doubt that these 
are the only three scenarios that may confuse the user while facing a feature implemented using 
Intents. I guess most users don't notice the switch and expect to not see the camera being used if 
they haven't explicitly allowed it. This questions the whole idea of using Intents though... Another
thing to note is that "revoked" and "was never requested" are probably two different states, which 
are however treated in the same way by the system. On Android 6, the user might not even know that 
the app will ask her to allow it to use the Camera at some point, therefore the use case of taking 
pictures via Intent will feel the same, independent of whether the permission has been declared in 
the Manifest or not, therefore the behavior of the system seems inconsistent to me. What's also 
disappointing is that articles that I read about runtime permissions, official and unofficial, 
present Intents as a "free alternative", and neither mentions these edge cases. I understand though,
that most apps will either use Intents, or declare `android.permission.CAMERA` and use the Camera 
directly, but the library use case is still a valid one.

Anyway, I had to deal with it, and the most straightforward solution I could think of was to add the
Camera permission into both apps - not ideal, since the other app doesn't in fact need it. Luckily, 
I stumbled upon a [nice solution][stackoverflow] on StackOverflow, suggesting to check if 
`android.permission.CAMERA` has been declared in the Manifest and then request it at runtime prior 
to firing the Intent. The code snippet looks like this:

```java
private static boolean hasPermissionInManifest(@NonNull Context context,
        @NonNull String permissionName) {
    String packageName = context.getPackageName();
    try {
        PackageInfo packageInfo = context.getPackageManager()
                .getPackageInfo(packageName, PackageManager.GET_PERMISSIONS);
        String[] declaredPermissions = packageInfo.requestedPermissions;
        if (declaredPermissions != null) {
            for (String p : declaredPermissions) {
                if (p.equals(permissionName)) {
                    return true;
                }
            }
        }
    } catch (PackageManager.NameNotFoundException e) {
        // ignored
    }
    return false;
}
```

Now, whenever I'd need to check if I have all necessary permissions granted, I could use the 
following code:

```java
public static boolean checkForPhotoPermissionsIfNeeded(@NonNull Fragment fragment,
        int requestCode) {
    boolean cameraPermissionDeclared = hasPermissionInManifest(fragment.getContext(),
        Manifest.permission.CAMERA);
    int storagePermissionStatus =
            ContextCompat.checkSelfPermission(fragment.getContext(), Manifest.permission.WRITE_EXTERNAL_STORAGE);
    int cameraPermissionStatus = !cameraPermissionDeclared ?
            PackageManager.PERMISSION_GRANTED :
            ContextCompat.checkSelfPermission(fragment.getContext(), Manifest.permission.CAMERA);
    if (storagePermissionStatus == PackageManager.PERMISSION_GRANTED &&
            cameraPermissionStatus == PackageManager.PERMISSION_GRANTED) {
        return true;
    }
    String[] permissionsToRequest = cameraPermissionDeclared ?
            new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE,
                    Manifest.permission.CAMERA} :
            new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE};
    fragment.requestPermissions(permissionsToRequest, requestCode);
    return false;
}
```

This definitely adds a number of test cases, but solves the problem and works fine in both apps.

### Conclusion

Be careful when using Intents, as sometimes they don't come for free, as you'd expect. Pay attention
to the documentation of Intent actions you use, especially in the context of Android M. Read this 
[amazing article][commonsware] by CommonsWare, which states that using `ACTION_IMAGE_CAPTURE` might 
actually be a bad idea. Test your code and love Android!

Cheers!

[taking-photos-simply]: https://developer.android.com/training/camera/photobasics.html
[permissions]: https://developer.android.com/training/permissions/best-practices.html#perms-vs-intents
[issue]: https://code.google.com/p/android/issues/detail?id=188073
[action-image-capture]: https://developer.android.com/reference/android/provider/MediaStore.html#ACTION_IMAGE_CAPTURE
[stackoverflow]: https://stackoverflow.com/questions/32789027/android-m-camera-intent-permission-bug
[commonsware]: https://commonsware.com/blog/2015/06/08/action-image-capture-fallacy.html
