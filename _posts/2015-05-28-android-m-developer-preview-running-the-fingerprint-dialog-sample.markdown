---
layout: post
title:  "Android M Developer Preview: Running the Fingerprint Dialog sample"
date:   2015-05-28 00:00:00 -0400
categories: android
---
So, the [Android M Developer Preview is out!][m-preview] All related info is already published, 
including the [Program Overview][program-overview], guide to [Setting up the SDK][setup-sdk] and the
collection of [Samples][samples] illustrating brand new Android features.

One of the hottest Android M features is definitely the 
[Fingerprint Authentication][fingerprint-authentication], which can be seen in action in a small 
[sample app][sample-app] ready to be downloaded from GitHub. So let's take a look!

First, make sure you've installed all necessary stuff to run Android M Developer Preview, and 
created an emulator with the newest Android version. Clone the repo, import the app into the Android
Studio and run it. Here's what we see:

{:refdef: style="text-align: center;"}
![oTmjLuu](/assets/oTmjLuu.jpg)
{: refdef}

To try the authentication, first we'll need to enroll a fingerprint. Navigate to **Settings > 
Security > Fingerprint** and follow the instructions to add one. Apparently, you'll need to set up 
either a PIN, a Pattern or a Password for the device in order to register a fingerprint. When done, 
you'll be presented with the following screen:

{:refdef: style="text-align: center;"}
![u52q6at](/assets/u52q6at.jpg)
{: refdef}

Tapping on the monitor probably won't get you far, so instead open the Terminal and use the 
following command to emulate the touch:

`adb -e emu finger touch 1`

'1' here identifies the fingerprint, so just use any random number value. The fingerprint should be
added successfully and you'll see the confirmation screen:

{:refdef: style="text-align: center;"}
![6nXX9LZ](/assets/6nXX9LZ.jpg)
{: refdef}

So far so good! Now let's head back to the sample app. The "Purchase" button is now available, so 
let's try to make a fingerprint-authenticated purchase:

{:refdef: style="text-align: center;"}
![CBOmGFi](/assets/CBOmGFi.jpg)
{: refdef}

Tap on "Purchase" and you'll be presented with the authentication dialog:

{:refdef: style="text-align: center;"}
![GS3EZah](/assets/GS3EZah.jpg)
{: refdef}

Open the Terminal again and type the same command used previously to register the fingerprint. The 
dialog will disappear - authentication successful!

Fingerprint Authentication promises to be a pretty interesting feature in the coming Android M 
release, but there's lots of other exciting stuff under the 
[Android M Developer Preview][m-preview]. Keep exploring!

[m-preview]: https://developer.android.com/preview/index.html
[program-overview]: https://developer.android.com/preview/overview.html
[setup-sdk]: https://developer.android.com/preview/setup-sdk.html
[samples]: https://developer.android.com/preview/samples.html
[fingerprint-authentication]: https://developer.android.com/preview/api-overview.html#authentication
[sample-app]: https://github.com/googlesamples/android-FingerprintDialog
