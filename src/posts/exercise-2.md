---
title: Exercise 2 - Ros Development and Kinematics
description: Exercise 2 - Ros Development and Kinematics
author: Marcus Huang
date: 2023-02-11T19:50:49.711Z
tags:
  - exercise
  - post
---

# Part 1

For the first part of the exercise, we mainly went through the tutorial and exercises in the duckietown handbook [\[1\]](#r1).

## Unit C-2.1 to C-2.6
### C-2.2, 2.3
by using the provided template, we created our first publisher node. The next section is just changing the content to include the `VEHICLE_NAME`.

![](/static/img/e2_c23.png "Output from the publisher node")

### C-2.4
We again create a subscriber node with the given code template that subscribes to our publisher node. We can visualize the relation with the `rqt_graph` command:

![](/static/img/e2_c24.svg "Graph generated with rqt_graph command")

### C-2.5
As we are adding more and more nodes, it is increasingly hard to handle everything in the `launch.sh` script. So this section introduces us to the `roslaunch` tool.

To use `roslaunch`, we create a `launch` subfolder in our package and create a launch file with the given template.

### C-2.6
One important thing when using a launch file is to understand namespaces and remapping since we describe a lot of relationships between the nodes so that they can communicate with each other. Also, the nodes are not necessarily all on the same bot, and we need a way to manage that.

Following the instruction, we add a group namespace in the launch file and run `rqt_graph` again to see the change:

![](/static/img/e2_ns.svg "Graph after namespace change")

We see that the two nodes are now under the `/VEHICLE_NAME` namespace.

Then, we edit the launch file to have 2 subscribers and 2 publishers:

![](/static/img/e2_multi.svg "rqt_graph after the change")

We see all the communication happening on the same topic, so we can't tell which node is the message from (or to).

To address that, we add a tilde sign before the topic name in the nodes. So that the node's name would be converted into a namespace, and the communication from each different node can take place in their own channel.

![](/static/img/e2_multi_chatter.svg "rqt_graph after the above change")

But we see that the nodes are not communicating because the subscriber nodes are also expecting the topics under their own namespaces.

We can remap the topics in the launch file to solve this.

Under the subscriber nodes, we add:
```
<remap from="~/chatter" to="/$(arg veh)/PUBLISHER/chatter"/>
```

![](/static/img/e2_multi_remap.svg "rqt_graph after the above change")

We see that the 2 pairs of nodes are communicating as intended now.

We replace the remapping with some other things to experiment with the namespace and remapping:

1. Replacing the destination of the remap in the subscriber node to `my_publisher_node_1/chatter`.

It still works, since both the subscriber and the publisher nodes are under the same `VEHICLE_NAME`, so the resolution of `my_publisher_node_1` would be done relative to the subscriber node's namespace. The result would be `/VEHICLE_NAME/my_subscriber_node_1/chatter`, and that topic exists.

It would be helpful if we use the analogy of how the URL or file path works.

2. Replacing the destination of the remap in the subscriber node to `/my_publisher_node_1/chatter`.

Now this will break the communication. Since the slash at the beginning indicates a root namespace, and there wasn't a `my_publisher_node_1` under the root.

3. We move the remap out of the subscriber section with the following code:

```
<remap from="my_subscriber_node_1/chatter" to="my_publisher_node_1/chatter"/>
<node pkg="my_package" type="my_subscriber_node.py" name="my_subscriber_node_1"  output="screen"/>
```

It works. Since the remap is under the `<group>` section and `my_subscriber_node_1` and `my_publisher_node_1` namespaces are under the same group (same parent namespace).

4. Again, remap out of the subscriber section but with the following code:
```
<remap from="~my_subscriber_node_1/chatter" to="~my_publisher_node_1/chatter"/>
<node pkg="my_package" type="my_subscriber_node.py" name="my_subscriber_node_1"  output="screen"/>
```
Though I didn't find anything directly explaining this (in ROS documentation [\[2\]](#r2)), and the `rqt_graph` didn't provide any extra information:

![](/static/img/e2_multi_rm4.svg "rqt_graph after the above change")

But I speculate that the `<group>` label isn't actually a node, therefore the tilde sign would not be able to convert the node's name into a namespace.

## Custom camera node
To implement a custom camera node, we need to first subscribe to the image topic (`/$(arg veh)/camera_node/image/compressed`) published by the driver node and read the image from it.

But the image message we received from the topic contains the compressed image (message type: `sensor_msgs/CompressedImage`) that can't be directly used, so we need to use `cv_bridge` module to convert the data into a CV2 image.

```
self.image = self._bridge.compressed_imgmsg_to_cv2(data)
```
(Note: import CV2 before cv_bridge, or Python interpreter will throw error)

With the image transformed into a CV2 image, we can now print out its size (640\*480\*3=921600):

![](/static/img/e2_multi_rm4.svg "my_camera_node is printing out the image size")

For the modification to the image, here I flipped it vertically.

![](/static/img/e2_cam_ori.svg "image from original camera topic")

![](/static/img/e2_cam_mod.svg "modified image from my_camera_node")

The code for the customized camera node can't fit in a single screenshot, please refer to the [repository](https://github.com/marcus65001/412e2/blob/v2/packages/my_package/src/my_camera_node.py).

## E-3.1
The exercises in this section are mostly about kinematics.

* What is the relation between your initial robot frame and world frame?

The initial robot frame describes the position of the robot in relation to itself while the world frame is in relation to its environment.

* How do you transform between them?

With initial and world frames defined, we can use the kinematics (initial -> world) and inverse kinematics (world -> initial) detailed in the eClass course notes [\[3\]](#r3) to derive transform matrices.
### Exercise 27 & 28
In the exercise, we learn how to obtain encoder information for both wheels and we need to modify the given template to add the following functionalities:

* Integrating the distance traveled by each wheel and publishing them under a new topic.
* Use a rosbag to record encoder ticks, distance traveled and wheel commands.

When using a rosbag:
* Make sure to close the bag properly. Rosbags will very likely be corrupted if not safely closed and reading from a corrupted bag will cause errors.
* On Duckiebots, since we are using dockers to run our code, the non-persistent data would be discarded when the container stops. Which means it would be very hard to obtain the bag files. So make sure to save the bags in persistent storage (like `/data`).

![](/static/img/e2_bag.png "bag messages")


[(Complete code)](https://github.com/marcus65001/412e2/blob/v2/packages/my_package/src/odometry_node.py)



### Straight line task
For this task, we need to drive our robot straight forward for 1.25 meters and then backward for 1.25 meters. Also, we need to keep track of our location with respect to our initial robot frame as well.

So how do we convert the location and theta at the initial robot frame to the world frame?

Looking at the definition, since the robot frame is fixed on the robot, the x, y coordinates will always be 0. So we don't actually use the location at the robot frame in the conversion. 

Instead, we use the change in x, y and theta at the robot frame and use kinematics to derive the transformation to obtain the change in these values at the world frame. And we integrate these changes over time, to get the final location and theta at the world frame.

Back to the actual task, I used the odometry node in Part 1 to keep track of the distance traveled and used keyboard control to move 1.25 meters forward. From the messages recorded in the rosbag, the robot thinks it has traveled 1.36m. So the desired location would be 1.36m forward, but the actual location is 1.25m forward.

I think the mismatch is caused by the random slippage on the track so that the actual distance moved is less than the wheel rotation.

To programmatically move the robot, I used the topic `kinematics_node/car_cmd`, with set velocity `v`. Since we use the keyboard control a lot, intuitively, I looked it up in the source code [\[4\]](#r4) and found out about this topic can be used to control the robot movement with given velocity input.

Alternatively, I've also looked into the topic `wheels_driver_node/wheels_cmd`, but that is sending raw wheels commands (`WheelsCmdStamped`), which does not take gain, trim and other parameters into consideration. If we write code to account for these, that would be repeating what has already been done in the `kinematics_node`.

I'm using 0.41 for the velocity since that's the default value used in the keyboard control (use `rostopic echo /VEHICLE_NAME/kinematics_node/velocity` to see what velocity is used).

![](/static/img/e2_bag.png "Twist2DStamped messages as seen in the kinematics_node/velocity topic")

Increasing the velocity makes the wheels move faster but also worsens the slippage problem. Also, it is capped by a `v_max` parameter, in the `kinematics_node` the velocity will be trimmed so that when it is over `v_max` increasing it would not have any effect.

Decreasing the velocity makes the wheels move slower, but also when the value is too small the robot might not move at all but we can hear motor sound.

### Rotation task
Similar to the straight line task, we again publish to the `kinematics_node/car_cmd` with velocity 0 and angular velocity `omega`.

To estimate the angle turned, we subscribe to the `kinematics_node/velocity` topic, which publishes the open loop estimate of current velocity. We take the angular velocity and integrate it over time, obtaining the total angle traveled.

### How to drive the robot autonomously in a circle?

Combining the above 2 tasks, we can set non-zero velocity and angular velocity to make the robot drive in a circle.

### Exit properly?
The task nodes will send a shutdown signal and exit upon completion using `rospy.signal_shutdown("done")`.

The odometry node will not exit and keep running until user interruption. Though I did not implement it in the node, it is possible to ping the task nodes in the odometry node with `rosnode.rosnode_ping("NODE_NAME")` [\[5\]](#r5) and shut down when all other nodes have exited.

# Part 2

## Keep track of the location
We use a dedicated `odometry_node` to keep track of the location.
The node subscribes to the `kinematics_node/velocity` topic, and constantly integrates the velocities over time, recording the x,y and theta coordinates in both the robot frame and world frame.

## LED Service
### First attempt
My first attempt to implement the LED service is to set up a service and directly publish the desired LED pattern to the `/hostname/led_driver_node/led_pattern` topic.

The first problem is to find out how to construct the message. The message type for the topic is `duckietown_msgs/LEDPattern`. As always, I can't really find anything about the structure of it. Again, we have to look at the source code.

From the dt-core source code [\[6\]](#r6):

```
from std_msgs.msg import ColorRGBA

rgba = ColorRGBA()
rgba.r = self.LEDspattern[i][0]
rgba.g = self.LEDspattern[i][1]
rgba.b = self.LEDspattern[i][2]
rgba.a = 1.0
LEDPattern_msg.rgb_vals.append(rgba)
```

So the LEDPattern uses `ColorRGBA` to describe the color pattern for each LED. In the same file, we can find out the mapping of LEDs on the duckiebot:

```
    Duckiebots have 5 LEDs that are indexed and positioned as following:
    +------------------+------------------------------------------+
    | Index            | Position (rel. to direction of movement) |
    +==================+==========================================+
    | 0                | Front left                               |
    +------------------+------------------------------------------+
    | 1                | Rear left                                |
    +------------------+------------------------------------------+
    | 2                | Top / Front middle                       |
    +------------------+------------------------------------------+
    | 3                | Rear right                               |
    +------------------+------------------------------------------+
    | 4                | Front right                              |
    +------------------+------------------------------------------+
```

### Change of plan
As I read through the source code, I found that the `led_emitter_node` has basically done what we wanted and it even provides a more simple approach to change the LED pattern with just the pattern name.

`led_emitter_node/set_pattern` service takes the pattern name as input and changes the LED. Predefined patterns can be seen in the [`LED_protocol.yaml`](https://github.com/duckietown/dt-core/blob/1cf2d69eb2900c66ef2b026546766cc5e648a721/packages/led_emitter/config/led_emitter_node/LED_protocol.yaml).

And my custom service node will provide `LEDSet` service which serves as a proxy to relay the pattern change to `led_emitter_node`.

Note: `led_emitter_node` has to be started before the custom service node with the following command.

```
dts duckiebot demo --demo_name led_emitter_node --duckiebot_name $BOT --package_name led_emitter --image duckietown/dt-core:daffy-arm64v8
```

Note 2: The "how to create and build msg and srv files" tutorial [\[8\]](#r8) on ROS wiki is not accurate:

In `CMakeLists.txt`, we need to add a few more dependencies as we used duckietown messages:
```
generate_messages(
  DEPENDENCIES
  std_msgs
  geometry_msgs
  sensor_msgs
  duckietown_msgs
)
```

In `package.xml`, `exec_depend` is deprecated. Use `run_depend` instead:
```
  <run_depend>message_runtime</run_depend>
```

## Task control node
The task control node divides the whole task into subtasks:
* Wait
* Change LED pattern
* Go forward
* Rotate clockwise
* Rotate counter-clockwise
* Pre-circle
* Circle

The task can then be described as a sequence of the above subtasks. A queue is used to keep track of the remaining subtasks. When executing movement subtasks the node subscribes to the related odometry topics to keep track of the progress. Another subroutine of the node will publish (or execute) up-to-date command according to the current state at an interval of 0.005 seconds (`rospy.Timer(rospy.Duration(0.005), node.pub_command)`).

### Wait
To implement the wait behavior, I used `rospy.get_time()` to record and compare the time elapsed. More details can be found on the [ROS wiki \[7\]](#r7).

### Turning behavior
One of the biggest challenges in this exercise is to implement a consistent turning behavior. Relying solely on the odometry information we have, it is extremely hard to perform turning consistently.

Through repeated testing, I found the following factors could be contributing to the inconsistency:

1. Error in odometry reading: The odometry reading is very inaccurate. As we are constantly doing integration and the difference between two consecutive messages is usually a very small float number, the numerical error will compound over time.
1. The swing: When the `steer_gain` is too high, the momentum will swing the robot a bit further past the stopping angle. The occurrence is almost completely stochastic.
2. The random slippage: Sometimes random slippage happens and the same amount of wheel rotation resulted in significantly less robot rotation than usual. There is no way to predict this and the only way I found to mitigate this is by reducing the gain. But that brings the next problem.
3. Stuck on the lane markers: The lane markers have a very different friction characteristic than the mat. Whenever (close to 80% of the time) the robot tries to rotate with any of the wheels touching the lane markers, the robot gets stuck. I can hear the motors constantly making a sound but not actually moving. I tried to set the `steer_gain` to the maximum possible value, but the problem persists. Through observing the turning behavior on other robots with similar steer gain value sets, I suspect that it might have been a hardware issue with my robot, maybe somehow the motors are underpowered. I tried to verify this on my teammate's robot, but it appears that it is having firmware issues.
4. Low update frequency: Initially, the node publishes commands at a 5 Hz frequency. But soon I discovered the update is too slow and combined with the inaccurate odometry reading, it will lead to bizarre turning behavior. Sometimes when it is already fairly close to the desired angle, but in the worst case the next command update to stop the turning will happen 0.2s later, so the robot will keep turning for the whole duration.

To mitigate the error in odometry reading, I tried to rewrite the odometry_node: instead of subscribing to `kinematics_node/velocity` that returns the velocity estimate, the node uses the raw wheels encoder information from `(left/right)_wheel_encoder_node/tick` to estimate the progress of the current task.

While implementing this, I found that the radius parameter set on my robot is not correct. According to datasheet [\[9\]](#r9), the diameter of the wheels is 66mm, therefore the radius would be roughly 0.207 (meters). The document confirmed that the resolution is indeed 135 and through a bit of calculation the baseline would be 0.102.

### End result
I have implemented State 1-2 and State 3-4 individually, but I haven't been able to combine the two parts in a satisfactory manner due to the aforementioned difficulties. I worked on the integration for an extra Saturday but unfortunately, the robot ran out of battery before I can complete a full run (that looks good on video).

Video for State 1-2:

https://youtu.be/e4u4SGL8fKo

Video for State 3-4:

https://youtu.be/Vz9lrv1AWrw

Video for rosbag print:

https://youtu.be/BjC8XhGon0M

Rosbag file:

[task.bag](https://github.com/marcus65001/cmput412/raw/main/exercise2/task.bag)

# Reference
<a name="r1">\[1\]</a> [Hands-on Robotics Development using Duckietown](https://docs.duckietown.org/daffy/duckietown-robotics-development/out/dt_infrastructure.html)

<a name="r2">\[2\]</a> [Names - ROS Wiki](http://wiki.ros.org/Names)

<a name="r3">\[3\]</a> eClass course notes

<a name="r4">\[4\]</a> [duckietown/dt-car-interface/joy_mapper_node.py](https://github.com/duckietown/dt-car-interface/blob/daffy/packages/joy_mapper/src/joy_mapper_node.py)

<a name="r5">\[5\]</a> [API to know if a node is alive? - ROS Answers: Open Source Q&A Forum](https://answers.ros.org/question/52457/api-to-know-if-a-node-is-alive/)

<a name="r6">\[6\]</a> [duckietown/dt-core/packages/led_emitter/src/led_emitter_node.py](https://github.com/duckietown/dt-core/blob/1cf2d69eb2900c66ef2b026546766cc5e648a721/packages/led_emitter/src/led_emitter_node.py)

<a name="r7">\[7\]</a> [rospy/Overview/Time - ROS Wiki](http://wiki.ros.org/rospy/Overview/Time)

<a name="r8">\[8\]</a> [ROS/Tutorials/CreatingMsgAndSrv - ROS Wiki](http://wiki.ros.org/ROS/Tutorials/CreatingMsgAndSrv#Creating_a_srv)

<a name="r9">\[9\]</a> [Duckiebot MOOC Founder's edition datasheet](https://docs.rs-online.com/c57a/A700000007343652.pdf)
