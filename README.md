# video-capture
Provide an RTSP stream for a connected video source.

## Description

The Capture Block takes a video source (usually a camera) as an input and converts it to an RTSP stream. It utilizes [gst-rtsp-server](https://github.com/GStreamer/gst-rtsp-server) for core functionality.

Input: The block will search for a Pi Camera (currently only on a Raspberry Pi) and use that by default. If it does not find a Pi Camera, it will look for a USB camera and use the first one it finds. If the camera supports YUYV it will use that, otherwise it will use mjpeg. You can override this automatic selection process by specifying your own Gstreamer pipeline using the service variable `GST_RTSP_PIPELINE`. 

You can also use a custom pipeline to change the default height, width, and framerate. For instance, here is the default pipeline used when a Pi camera is detected:

`rpicamsrc bitrate=8000000 awb-mode=tungsten preview=false ! video/x-h264, width=640, height=480, framerate=30/1 ! h264parse ! rtph264pay name=pay0 pt=96`

To change the height and width when using a pi Camera, edit the values in the above pipeline and then set the `GST_RTSP_PIPELINE` to the edited pipeline.

For a standard USB webcam that uses YUYV:

`v4l2src device=/dev/video0 !  v4l2convert ! video/x-raw,width=640,height=480,framerate=15/1 ! omxh264enc target-bitrate=6000000 control-rate=variable ! video/x-h264,profile=baseline ! rtph264pay name=pay0 pt=96`

On the Jetson Nano it would be:

`v4l2src device=/dev/video0 ! nvvidconv ! video/x-raw(memory:NVMM) ! omxh264enc bitrate=6000000 control-rate=variable ! video/x-h264,profile=baseline ! rtph264pay name=pay0 pt=96`

For an older webcam that only supports jpeg:

`v4l2src device=/dev/video0 ! image/jpeg,width=640,height=480,framerate=10/1 ! queue ! jpegdec ! omxh264enc target-bitrate=6000000 control-rate=variable ! video/x-h264,profile=baseline ! rtph264pay name=pay0 pt=96`

In fact, you can change almost any part of the pipeline to suit your needs, but make sure it uses the proper device name and ends with an rtp output such as `rtph264pay`. Also be sure that your camera (or source) supports the desired size and framerate. When the capture block starts, it displays this information. You can also ssh into the block and run `v4l2-ctl --list-formats-ext` to see supported features.

**A RTSP stream will be available on `rtsp://localhost:8554/server` to other containers in the application. (Replace localhost with the device's IP address to view the stream outside the device)**

## Usage
To use this image, create a container in your `docker-compose.yml` file as shown below:
```
version: '2'

services:
  video-capture:
    build: .
    network_mode: host
    privileged: true
    labels:
          io.balena.features.balena-api: '1'
```

The capture block will automatically set `BALENA_HOST_CONFIG_start_x` to `1` and `BALENA_HOST_CONFIG_gpu_mem` to `192`. This effectively utilizes 192 MB of system memory for the GPU, regardless of your dashboard setting for "Define device GPU memory in megabytes." Note that when the block changes these settings, your device will reboot. (Usually, this only happens the first time the block runs.)

## Compatibility
This block has been tested on the following devices:
- Raspberry Pi 3B+
- Raspberry Pi 4 (2GB)
- Raspberry Pi Zero W (limited frame rate)
- Nvidia Jetson Nano 

The following cameras have been tested:
- Microsoft LifeCam Cinema
- Microsoft LifeCam VX-3000
- Raspberry Pi Camera Module v2

## Troubleshooting
Certain (mostly older) webcam's YUYV mode may not be supported by the v4l2 plugin being used by this block. If this is the case, you won't get any video and may see errors similar to the following in the log window:
```
<video-capture> 37   0x264bb0 WARN               rtspmedia rtsp-media.c:1834:default_handle_message: 0xf6a3e480: got error Internal data stream error. 

(gstbasesrc.c(3055): gst_base_src_loop (): /GstPipeline:media-pipeline/GstBin:bin3/GstV4l2Src:v4l2src3:
<video-capture> streaming stopped, reason not-negotiated (-4))
<video-capture> 37   0x281890 ERROR             rtspclient rtsp-client.c:678:find_media: client 0x161ae0: can't prepare media
```
As a workaround, you can use a custom pipline (like the one listed above) to force the camera into jpeg mode, if supported. You could also try adding the following [configuration variable](https://www.balena.io/docs/learn/manage/variables/) to the dashboard: `GST_V4L2_USE_LIBV4L2` and set its value to `1`. This will enable the depricated libv4l2 library which may have broader conversion ability.
