
---

## The setup

I run a ROS 2 [CHECK: Humble?] localization stack on an outdoor mobile robot: a Livox Mid-360
providing 3D LiDAR and built-in IMU, feeding FAST-LIO for LiDAR-inertial odometry, with
`robot_localization` fusing that against wheel odometry. Standard enough arrangement.

The robot itself has a large footprint and works in tight spaces, so the costmap has to be right.
When it isn't, you find out quickly.

## The symptom

The point cloud looked plausible in RViz at a glance — right density, right range, nothing obviously
broken. But obstacles appeared on the wrong side of the robot. A wall on the left showed up in the
costmap on the right.

[CHECK: describe what you actually saw first. Was it the costmap, an unexpected collision-monitor
stop, a navigation abort, or did you spot it in RViz? The concrete first symptom is the most useful
part of the post for anyone searching later.]

What made it slow to diagnose: nothing errored. No warnings, no TF timeouts, no dropped messages.
Every node reported healthy. The cloud was geometrically self-consistent — it was just reflected
about [CHECK: which axis?].

## The investigation

What I did, roughly in order:

1. **Recorded a rosbag** in the failing condition so I could stop debugging on a live robot in a
   field and start debugging at a desk. This is the step I'd skip if I were in a hurry, and the
   step that saves the most time.
2. **Checked the raw cloud against the transformed cloud.** Visualising the points in the sensor
   frame rather than the map frame showed the raw data was fine. That immediately moved the problem
   out of the sensor and into the transform chain.
3. **Walked the TF tree.** `ros2 run tf2_tools view_frames` plus `tf2_echo` on the sensor-to-base
   link. The tree was *valid* — connected, no loops, no missing links — which is precisely why
   nothing complained.
4. **Compared the transform actually being published against the one I expected.** That's where it
   fell out.

## The root cause

The sensor extrinsics were declared in two places, and both were being applied.

The Livox driver takes extrinsic parameters in its own configuration [CHECK: confirm the exact
file/field names for your driver version before posting — `livox_ros_driver2` uses an extrinsic
block in its config JSON, but name it precisely or describe it generically]. The same
sensor-to-base transform was *also* present in the URDF as a joint between the base link and the
sensor link.

So the mounting rotation was composed twice. [CHECK: state the actual numbers if you're comfortable
— e.g. a 180° yaw applied in both places nets 360°, whereas a 90° applied twice gives 180°. The
arithmetic explains cleanly why the result is mirrored rather than merely rotated, and readers will
want it.]

Neither source was wrong on its own. Neither could detect the other. The result was a TF tree that
was internally consistent and externally wrong — the worst kind, because every diagnostic tool you'd
reach for reports success.

## The fix

Pick one owner for the extrinsic and make the other one identity.

I kept the URDF as the single source of truth and zeroed the extrinsic in the driver configuration
[CHECK: confirm this is the direction you chose — if you did it the other way round, say so and say
why]. The URDF is where the rest of the robot's geometry lives, it's version-controlled alongside
everything else, and it's what people go to look at when they want to know where a sensor is.

## What I'd tell anyone hitting this

- **A valid TF tree is not a correct TF tree.** `view_frames` confirms connectivity and nothing
  more. If your frames connect and your data is still wrong, the transform values are the suspect,
  not the structure.
- **Any driver with its own extrinsic configuration is a duplication risk** the moment that sensor
  also appears in a URDF. Livox drivers, several camera drivers, some IMU drivers. Worth an explicit
  check whenever you add a sensor.
- **Mirrored, not rotated, is a fingerprint.** A cloud that's rotated by an odd amount usually means
  a wrong extrinsic. A cloud that's cleanly mirrored or double-rotated often means a *doubled* one.
- **Decide where extrinsics live before you have two of them,** and write it down. This is a
  five-minute convention that prevents a multi-day bug.



