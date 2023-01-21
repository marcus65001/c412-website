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

https://www.youtube.com/watch?v=VjdxAHQwB48

But through repeated testing, I found that the drift pattern is actually random, and it does not appear to be consistently drifting to one side. Upon closer inspection, I suspected that it is because of the lanes are not perfectly flat and the presence of debris. When the wheels hit debris and the gaps between the tiles, the robot will sometimes be driven off the original track, causing the random drifting pattern.

#### C﻿amera