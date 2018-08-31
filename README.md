---
page: https://idle.run/opencv-mjpeg-fps
title: "OpenCV V4L High FPS MJPEG Support"
tags: opencv c++ v4l
date: 2018-08-31
---

## Overview

The regular linux build of OpenCV 3.4.2 has MJPEG support disabled in the V4L video source implementation.

This post details how to enable that support.

## Environment

Install the V4L utilities to confirm device info

`apt install v4l-utils`

`v4l2-ctl --list-formats-ext -d /dev/video1`

```
ioctl: VIDIOC_ENUM_FMT
  Index       : 0
  Type        : Video Capture
  Pixel Format: 'MJPG' (compressed)
  Name        : Motion-JPEG
    Size: Discrete 1920x1080
      Interval: Discrete 0.033s (30.000 fps)
    Size: Discrete 1280x720
      Interval: Discrete 0.017s (60.000 fps)
    Size: Discrete 1024x768
      Interval: Discrete 0.033s (30.000 fps)
    Size: Discrete 640x480
      Interval: Discrete 0.008s (120.101 fps)
    Size: Discrete 800x600
      Interval: Discrete 0.017s (60.000 fps)
    Size: Discrete 1280x1024
      Interval: Discrete 0.033s (30.000 fps)
    Size: Discrete 320x240
      Interval: Discrete 0.008s (120.101 fps)

  Index       : 1
  Type        : Video Capture
  Pixel Format: 'YUYV'
  Name        : YUYV 4:2:2
    Size: Discrete 1920x1080
      Interval: Discrete 0.167s (6.000 fps)
    Size: Discrete 1280x720
      Interval: Discrete 0.111s (9.000 fps)
    Size: Discrete 1024x768
      Interval: Discrete 0.167s (6.000 fps)
    Size: Discrete 640x480
      Interval: Discrete 0.033s (30.000 fps)
    Size: Discrete 800x600
      Interval: Discrete 0.050s (20.000 fps)
    Size: Discrete 1280x1024
      Interval: Discrete 0.167s (6.000 fps)
    Size: Discrete 320x240
      Interval: Discrete 0.033s (30.000 fps)
```

## Problem

After building OpenCV from source (with V4L support). I could only get the device to load in the YUYV formats listed above.

Tried setting the FPS with OpenCV commands and setting to MJPG mode

```
  cap->set(cv::CAP_PROP_FPS, CAMERA_FPS);
  cap->set(cv::CAP_PROP_FRAME_WIDTH, CAMERA_WIDTH);
  cap->set(cv::CAP_PROP_FRAME_HEIGHT, CAMERA_HEIGHT);
  cap->set(cv::CAP_PROP_FOURCC, CV_FOURCC('M','J','P','G'));
```

But retrieving the settings showed that they were not applied (30 FPS was returned for 640x480)

```
cap->get(cv::CAP_PROP_FPS)
```

## What's Wrong

Internally there are actually *two* `VideoCapture` implementations in OpenCV:

https://github.com/opencv/opencv/blob/master/modules/videoio/src/cap_v4l.cpp

https://github.com/opencv/opencv/blob/master/modules/videoio/src/cap_libv4l.cpp

For some unknown reason, the cap_libv4l version does *not* support MJPEG mode.

IE compare:

https://github.com/opencv/opencv/blob/master/modules/videoio/src/cap_libv4l.cpp#L737

with

https://github.com/opencv/opencv/blob/master/modules/videoio/src/cap_v4l.cpp#L430


## Solution

Thankfully the solution is pretty simple, just a matter of modifying the `cvconfig.h` generated during the OpenCV build

After running `cmake`, edit `cvconfig.h` to *remove* the definition of `HAVE_LIBV4L`.
The `cap_v4l` implementation is enabled when `HAVE_CAMV4L2` is present and `HAVE_LIBV4L` is not.

```
cmake  -D CMAKE_BUILD_TYPE=RELEASE \
       -D CMAKE_INSTALL_PREFIX=/usr/local \
       -D WITH_CUDA=ON -D CUDA_ARCH_BIN="6.2" -D CUDA_ARCH_PTX="" \
       -D WITH_CUBLAS=ON -D ENABLE_FAST_MATH=ON -D CUDA_FAST_MATH=ON \
       -D ENABLE_NEON=ON -D WITH_LIBV4L=ON -D BUILD_TESTS=OFF \
       -D BUILD_PERF_TESTS=OFF -D BUILD_EXAMPLES=OFF \
       -D WITH_QT=ON -D WITH_OPENGL=ON ..

sed -i -e 's/\(#define HAVE_LIBV4L\)/\/\/\1/' cvconfig.h

make

make install
```

After that the exact same code above works as expected in MJPEG mode.
