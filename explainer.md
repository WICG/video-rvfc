# video.requestVideoFrameCallback() Explainer

# Introduction
Today [`<video>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement) elements have no means by which to signal when a video frame has been presented for composition nor any means to provide metadata about that frame.

We propose a new HTMLVideoElement.requestVideoFrameCallback() method with an associated VideoFrameRequestCallback to allow web authors to identify when and which frame has been presented for composition.


# Use cases

Today sites using [WebGL](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API) or [`<canvas>`](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) with a [`<video>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement) element rely on a set of heuristics to determine if the first frame has been presented ([video.currentTime > 0](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/currentTime), video [canplaythrough](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/canplaythrough_event) event, etc). These heuristics vary across browsers. For subsequent frames sites must blindly call [Canvas.drawImage()](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawImage) or [GLContext.texImage2D()](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texImage2D) for access to the pixel data. Our proposed API would allow reliable access to the presented video frame.

In an era of [Media Source Extensions](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API) based [adaptive video playback](https://en.wikipedia.org/wiki/Adaptive_bitrate_streaming) (e.g., [YouTube](https://www.youtube.com/), [Netflix](https://www.netflix.com/), etc) and boutique [WebRTC](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) streaming solutions (e.g., [Rainway](https://rainway.com/), [Stadia](https://store.google.com/us/magazine/stadia)), the inherent raciness of access operations and the lack of metadata exposed about the frames limits quality and automated analysis. Our proposed API would allow reliable access to metadata about which frame has been presented when for correlation with other page level events (user input, etc).

Additionally, our proposal will enable will enable a host of frame-accurate [web-platform-tests](https://github.com/web-platform-tests/wpt) which can be shared across browsers that were heretofore impossible or otherwise flaky. E.g., while the [HTMLMediaElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement) spec defines [readyStates](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/readyState) in terms of buffering, it does not define when a frame will be present on the screen.

Specific use case examples:
* WebGL applications would like to composite at the video rate and not the display rate to save on processing complexity.
* WebRTC applications would like to synchronize user events such as a key press with the frame that was displayed to the user when the event was triggered.


# Proposed API

```Javascript
dictionary VideoFrameMetadata {
    // The time at which the user agent submitted the frame for composition.
    required DOMHighResTimeStamp presentationTime;

    // The time at which the user agent expects the frame to be visible.
    required DOMHighResTimeStamp expectedDisplayTime;

    // The width and height of the presented video frame.
    required unsigned long width;
    required unsigned long height;

    // The media presentation time in seconds of the frame presented. This
    // should match the value of `video.currentTime` when the frame is displayed.
    required double mediaTime;

    // The elapsed time in seconds from submission of the encoded packet with
    // the same presentationTimestamp as this frame to the decoder until the
    // decoded frame was ready for presentation.
    //
    // In addition to decoding time, may include processing time. E.g., YUV
    // conversion and/or staging into GPU backed memory.
    double processingDuration;  // optional

    // A count of the number of frames submitted for composition. Allows clients
    // to determine if frames were missed between VideoFrameRequestCallbacks.
    //
    // https://wiki.whatwg.org/wiki/Video_Metrics#presentedFrames
    required unsigned long presentedFrames;

    // For video frames coming from either a local or remote source, this is
    // the time at which the frame was captured by the camera. For a remote
    // source, the capture time is estimated using clock synchronization and
    // RTCP sender reports to convert RTP timestamps to capture time as
    // specified in RFC 3550 Section 6.4.1.
    DOMHighResTimeStamp captureTime;  // optional

    // For video frames coming from a remote source, this is the time the
    // encoded frame was received by the platform, i.e., the time at which the
    // last packet belonging to this frame was received over the network.
    DOMHighResTimeStamp receiveTime; // optional

    // The RTP timestamp associated with this video frame.
    //
    // https://w3c.github.io/webrtc-pc/#dom-rtcrtpcontributingsource
    unsigned long rtpTimestamp;  // optional
};

callback VideoFrameRequestCallback = void(DOMHighResTimeStamp time, VideoFrameMetadata);

partial interface HTMLVideoElement {
    unsigned long requestVideoFrameCallback(VideoFrameRequestCallback callback);
    void cancelVideoFrameCallback(unsigned long handle);
};
```


# Example

```Javascript
  let video = document.createElement('video');
  let canvas = document.createElement('canvas');
  let canvasContext = canvas.getContext('2d');

  let frameInfoCallback = (_, metadata) => {
    console.log(
      `Presented frame ${metadata.presentationTimestamp}s ` +
      `(${metadata.width}x${metadata.height}) at ` +
      `${metadata.presentationTime}ms for display at ` +
      `${expectedPresentationTime}ms`);

    canvasContext.drawImage(video, 0, 0, metadata.width, metadata.height);
    video.requestVideoFrameCallback(frameInfoCallback);
  };

  video.requestVideoFrameCallback(frameInfoCallback);
  video.src = 'foo.mp4';
```

Output:
```Text
Presented frame 0s (1280x720) at 1000ms for display at 1016ms.
```


# Implementation Details
* `video.requestVideoFrameCallback()` callbacks are one-shot, and must be called again to get the next frame.
* Since `VideoFrameRequestCallback` will only occur on new frames, error states may never satisfy calls to `requestVideoFrameCallback`.
* In cases where `VideoFrameMetadata` can't be surfaced (e.g., [encrypted media](https://w3c.github.io/encrypted-media/#media-element-restrictions)) implementations may never satisfy calls to `requestVideoFrameCallback`.
* `VideoFrameRequestCallbacks` are run before `window.requestAnimationFrame()` callbacks, during the "[update the rendering](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering)" steps.
* `window.requestAnimationFrame()` callbacks registered from within a `video.requestVideoFrameCallback()` callback will be run in the same turn of the event loop. E.g:
```Javascript
  video.requestVideoFrameCallback(vid_now => {
    window.requestAnimationFrame(win_now => {
        if (vid_now != win_now)
            throw "This should never throw";
    });
  });
```
* The rate at which `VideoFrameRequestCallbacks` are run is the minimum between the video rate and
browser rate. This is because they don't fire more than once per new frame, and more than once per
rendering steps. For example:

| Video rate | Browser rate | Callback rate |
|---|---|---|
| **25fps** | 60hz | 25hz |
| 60fps | **30hz** | 30hz |
| **60fps** | **60hz** | 60hz |
| 120fps | **60hz** | 60hz |


# Open Questions / Notes / Links
* [Link to GitHub repository.](https://github.com/WICG/video-raf)
* [Link to WICG Discourse.](https://discourse.wicg.io/t/proposal-video-requestanimationframe/3691)
* [Link to TAG review.](https://github.com/w3ctag/design-reviews/issues/429)
