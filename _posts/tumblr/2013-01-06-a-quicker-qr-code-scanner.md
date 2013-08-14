---
layout: post
title: 'A Quicker QR Code Scanner '
tags: 
---

[QR codes](http://en.wikipedia.org/wiki/QR_code) have become very popular in the advertising industry. They are slapped onto flyers, business cards and many other marketing materials with the hope that consumers will scan them. QR codes sound like a great idea in writing, but they are faced with many problems being adopted in the real world. One of these problems is the neglect of QR codes by smartphone manufacturers. There are a plethora of applications available for each given smartphone to scan QR codes, but the fact that they are third party detracts from their value. They don’t come pre-installed, involve hassle to activate, and each one is different, causing fragmentation. Countless times have I seen a QR code only to be deterred by the fact that I must first unlock my iPhone, navigate to a page of rarely used applications, open a QR code scanner, and wait for the camera to initialize. If smartphone manufacturers adopted QR codes and integrated scanning functionality into their software, all of these problems would be solved. In Japan, for example, QR codes are immensely popular and almost all phones have QR code scanning functionality. I don’t know which one caused the other, but it does seem as if those two things go hand in hand. In the US, the only smartphone (that I know of) that has QR code scanning functionality built in is Windows Phone, with [Bing Vision search](http://www.windowsphone.com/en-us/how-to/wp7/web/scan-codes-tags-and-text). Google also has a visual search platform, [Google Goggles](http://en.wikipedia.org/wiki/Google_Goggles), with apps on iOS and Android, but the functionality has not yet been integrated into Android itself.

Interestingly enough, Apple has adopted 2D barcodes (including QR codes) in its new Passbook service in iOS 6. By doing this, Apple is expressing confidence that 2D barcodes will continue to be relevant for a long time to come. It would only make sense for them to integrate QR code scanning into iOS. As some have [suggested](http://techcrunch.com/2012/09/01/how-apple-and-google-could-make-qr-codes-mainstream/), this could be done by making the camera application a bit smarter, to automatically look out for QR codes. That article does get one thing wrong, though, in saying that the idea is “probably not as simple to implement as it seems”. As it turns out, the idea *is* simple to implement, at least on iOS.

## Analyzing The Camera

In order to integrate QR code scanning functionality into the camera, I first needed to understand how the camera works. I did this by reverse engineering Apple’s software. I discussed popular tools that I use to reverse engineer software in my previous [blog post](http://kramerapps.com/blog/post/38090565883/integrate-cloud-print-ios). The framework that controls basic camera interactions on iOS is the private framework PhotoLibrary. The framework includes the view controllers that display the Camera Roll and also the controller that is responsible for taking photos, `PLCameraController`. These controllers are not only used in the Camera application, but are also used to back the public `UIImagePickerController` class. The PhotoLibrary framework is based on the public AVFoundation framework. Here is the interface for `PLCameraController`, abridged to only show the parts we care about:

``` objc
@interface PLCameraController : NSObject {
    AVCaptureDeviceInput *_avCaptureInputFront;
    AVCaptureDeviceInput *_avCaptureInputBack;
    AVCaptureDeviceInput *_avCaptureInputAudio;
    AVCaptureStillImageOutput *_avCaptureOutputPhoto;
    AVCaptureMovieFileOutput *_avCaptureOutputVideo;
    AVCaptureVideoDataOutput *_avCaptureOutputPanorama;
}

+ (instancetype)sharedInstance;

@property (weak, nonatomic) AVCaptureOutput *currentOutput;
@property (weak, nonatomic) AVCaptureDeviceInput *currentInput;
@property (readonly, strong, nonatomic) AVCaptureSession *currentSession;
@property (readonly, strong, nonatomic) AVCaptureVideoPreviewLayer *previewLayer;

@end
```

As you can see, the basic structure of the camera controller is an `AVCaptureSession` with a number of inputs and outputs. The microphone, front and back cameras function as inputs, and there are outputs for taking photos, videos, and panorama shots. When I first looked into the PhotoLibrary framework before iOS 6 was released, these observations led me to [discover](http://www.wired.com/gadgetlab/2011/11/enable-secret-panorama-feature-in-ios5/) the then unreleased Panorama mode.

## Adding QR Code Scanning

Once I knew the basic structure of the PhotoLibrary framework, I was able to work on integrating a QR code scanner into it. The library I used to read QR codes was [ZXing](http://code.google.com/p/zxing/), pronounced “zebra crossing”. It is a popular open source library for interpreting 2D barcodes, originally written in Java, and then ported to C++. There are also Objective-C wrappers available in the source distribution (two of them, in fact), but I found both heavy-handed for my purposes. In addition, both Objective-C wrappers process image data on the main thread, which I found to cause noticeable interface lag. I chose instead to write a small class (100 lines) that implements the functionality I need. The class I wrote takes data from a `AVCaptureVideoDataOutput` and processes it with the ZXing C++ library on a private queue.

Thanks to Apple’s straightforward [media capture API](https://developer.apple.com/library/ios/#documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/04_MediaCapture.html) in AVFoundation, it is easy to add arbitrary outputs to a capture session. All I needed to do was create an instance of my class, and add the video data output to the session in the camera controller, and just like that, I was recognizing QR codes. The implementation was just as straightforward as the idea. I made something to read QR codes, and quite simply added it into the Camera. Don’t believe me? Check out the [source code](https://github.com/conradev/QuickQR)! I also made a video, demonstrating the tweak in action:

<iframe width="560" height="315" src="http://www.youtube.com/embed/HQNB9XZdPCk" frameborder="0" allowfullscreen></iframe><br/>

If you are interested in tweaking iOS, be sure to check out my previous [blog post](http://kramerapps.com/blog/post/38090565883/integrate-cloud-print-ios) for a more in-depth discussion. Feel free to [follow me](http://twitter.com/conradev) on Twitter or check out my [software](http://kramerapps.com). Additionally, if your company is hiring mobile dev interns this upcoming summer, and has no inhibitions hiring a high school student, be sure to [get in touch](conrad@kramerapps.com).

Discuss on Hacker News [here](http://news.ycombinator.com/item?id=5016497).