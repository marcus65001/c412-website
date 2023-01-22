---
title: Exercise 1 - Duckiebot Assembly and Basic Development
description: Exercise 1 - Duckiebot Assembly and Basic Development
author: Marcus Huang
date: 2023-01-21T19:50:49.711Z
tags:
  - exercise
  - post
---
## Setup

The Duckiebot I received:

![](/static/img/selfie.jpg)

Logging into the dashboard, we can see the motor signals and the camera feed:

![](/static/img/dashboard.png)

### Calibration

#### W﻿heels

The Duckiebot uses a differential drive, so theoretically, to drive in a straight line, we turn the motors at the same speed. But due to the wear of the wheels and motors, in reality, the wheels turn differently given the same signal. So we need to recalibrate to account for this.

Following the [instructions](https://docs.duckietown.org/daffy/opmanual_duckiebot/out/wheel_calibration.html):

In the first test, the robot drifted to the left:

https://www.youtube.com/watch?v=MoCdTmNNuaM

But through repeated testing, I found that the drift pattern is actually random, and it does not appear to be consistently drifting to one side. Upon closer inspection, I think that it is because of the lanes are not perfectly flat and the presence of debris. When the wheels hit debris and the gaps between the tiles, the robot will sometimes be driven off the original track, causing the random drifting pattern.

![](/static/img/calibration_stray_right.jpg "Robot drifting to the right")

#### C﻿amera

For both calibration processes, we use the [calibration chequerboard](https://github.com/duckietown/duckietown-mplan/blob/a025c99e218687685d80bd72a7f90996572a55c7/hardware/camera_calibration_pattern_A3.pdf).

##### Intrinsic

This process is to account for discrepancies between different cameras.

Following the instruction, we do the "calibration dance":

![](/static/img/calibration_int.png)

![](/static/img/cam_int.png)

```
camera_matrix:
  cols: 3
  data:
  - 293.9407611716643
  - 0.0
  - 308.82892584255376
  - 0.0
  - 290.91876084253977
  - 244.40635847540514
  - 0.0
  - 0.0
  - 1.0
  rows: 3
camera_name: csc22909
distortion_coefficients:
  cols: 5
  data:
  - -0.2382383464120356
  - 0.03851835185678918
  - -0.004198322888723844
  - 0.0005723467308736188
  - 0.0
  rows: 1
distortion_model: plumb_bob
image_height: 480
image_width: 640
projection_matrix:
  cols: 4
  data:
  - 200.7656707763672
  - 0.0
  - 309.6695870311505
  - 0.0
  - 0.0
  - 227.88687133789062
  - 240.35899712785613
  - 0.0
  - 0.0
  - 0.0
  - 1.0
  - 0.0
  rows: 3
rectification_matrix:
  cols: 3
  data:
  - 1.0
  - 0.0
  - 0.0
  - 0.0
  - 1.0
  - 0.0
  - 0.0
  - 0.0
  - 1.0
  rows: 3
```

##### Extrinsic

This process is to account for the positioning relative to the environment. It can be done by placing the robot like this:

![](/static/img/calibration_ext.jpg)

The result:

![](/static/img/cam_ext.png)

```
homography:
- -5.387951517925476e-05
- -0.0001923101891734577
- -0.14083050325513338
- 0.0008351887687183021
- -2.3499620425059767e-05
- -0.2628749079198232
- -0.00014556892040089637
- -0.005834035789319427
- 1.0

# Calibrated on 2023-01-13
```

### Networking

Going through the exercises:

We can find MAC addresses and find out about the subnetwork information with the `ifconfig` command.

Sidenote: The notation in the handbook (`192.168.1.xyz`) is not exactly accurate, this is only true when the prefix length is 24, so that the range is 1 to 255. We can see that when the subnet is `192.168.1.0/25`, then the subnet is actually ranging from `192.168.1.1` to `192.168.1.127`. In this sense, sticking with the CIDR notation might be more accurate.

With default configuration under Ubuntu, plain hostnames are not resolved (apart from our own machine), we use `hostname.local` to reach local devices.

For the SSH part, we can follow the instructions in the cheat sheet for creating SSH key and SSH into the Duckiebot sections.

## Lane Following

Gone through the Docker basics, now it is time to try out the lane-following demo. Confirming the required containers are all running, we start the lane_following demo:

https://youtu.be/_IyGGHic2gc

I have also attempted the optional activity (visualize the detected line segments) in the handbook, but it appears that the instructions are dated and the `line_detector_node` does not publish the `image_with_lines` topic anymore. So I recorded the next closest visualization:

https://www.youtube.com/watch?v=7dM2w53VJrE

## Git and Running Python

For this part, I cloned the [template](https://github.com/duckietown/template-basic/) provided, made necessary changes (Dockerfile, launch script and add dependencies), and created a new Python package.

![](/static/img/python_on_duckie_numpy.png)

For some reason, the handbook gave the build command for `arm32v7` architecture, we change that to the correct platform (which is actually `arm64v8`).

## More Docker

For B-3.1 Exercise 13, we need to mount our home directory in the container's home directory. Copying the command in the handbook resulted in the following error:

`docker: Error response from daemon: create ~: volume name is too short, names should be at least two alphanumeric characters.`

Resolved by using the full path to the home directory.

The next part is creating a functional Docker container for colour detection:

The specification for the colour detector given in the handbook is rather vague: by "most present color", do they mean our detector should output the exact colour values of whatever colour that appears the most by frequency count or within a predefined set of colours we detect whether any of these are present in the section?

As it is to be implemented for Duckiebots, I assume the latter is more useful in this sense and thus the set of colour selected would naturally be white, red and yellow, which are the colours we used to mark different features on the Duckietown lanes.

When implementing the colour detection:

* The output we got from the camera stream is in the BGR colour space. But it is easier to define colour ranges that can take environment changes (illumination) into account in HSV colour space. So I first convert the frames into HSV.

  * [OpenCV documentation](https://docs.opencv.org/4.x/df/d9d/tutorial_py_colorspaces.html) explained this in detail
* In HSV, the colour red is on two ends of the hue spectrum, so two sets of ranges are needed to properly capture red colour.

The final colour detection module implemented ([Docker Hub](https://hub.docker.com/r/marcus65001/colordetector), [Github](https://github.com/marcus65001/duckie_color_det)) would take camera stream and the n_split environment parameter as input and output a table of most present colour of interest detected in the divided n horizontal areas:

![](/static/img/color_det.png "Example output on test image")

The Docker image built without issue, but it is having trouble opening the camera device.

![](/static/img/gst_fail.png)

To resolve this, I attempted:

* As mentioned in the handbook, we might need to mount the `argus_socket` volume to use the GST pipeline. I got the same error with the new command.
* Also in the handbook, we need to make sure no other container should access the camera at the same time. So I rebooted the robot, checked and stopped all containers that have camera access. Issue persists.
* Handcraft the GST pipeline string using the set of parameters that are assumed to be working in the camera [driver code](https://github.com/duckietown/dt-duckiebot-interface/blob/daffy/packages/camera_driver/src/jetson_nano_camera_node.py).

  * Parameters tried:

    * Camera Modes

      ```
      CameraMode(0, 3264, 2464, 21, "full")
      CameraMode(1, 3264, 1848, 28, "partial")
      CameraMode(2, 1920, 1080, 30, "partial")
      CameraMode(3, 1640, 1232, 30, "full")
      CameraMode(4, 1280, 720, 60, "partial")
      CameraMode(5, 1280, 720, 120, "partial")
      ```
    * HW acceleration on/off
* I also tried to use the GST string used in the `duckiebot-interface` node, through inspecting the logs in the Portainer, we get the following:

  * ```
    nvarguscamerasrc
    sensor-mode=3 exposuretimerange="100000 80000000" !
    video/x-raw(memory:NVMM), width=640, height=480, format=NV12, framerate=30/1 !
    nvjpegenc !
    appsink
    ```

None of the attempts resolves the issue.

Though probably out of the scope of this specific part of exercise, I think it is possible to resolve the issue by getting the camera output from the topics published by `camera_node`.