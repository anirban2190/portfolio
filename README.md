# Debugging a mirrored point cloud: extrinsics applied twice between driver config and URDF

*Draft for ROS Discourse — category suggestion: General / Localization / Sensor Integration*

> **Before posting — fill in or verify the items marked `[CHECK]`.** I've written this from the
> outline of your CV, so the shape of the story is right but several specifics are placeholders.
> Getting one of them wrong in public is worse than not posting.
>
> **IP safety:** there is no employer code here, no config file contents, no product details, and
> no customer named. The setup is described generically because the bug is generic. Keep it that
> way — if you add a snippet, retype it as a minimal reproduction rather than pasting your actual
> config.

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

## Question for the community

Does anyone enforce this structurally rather than by convention? I've considered a startup check
that compares the driver's configured extrinsic against the corresponding URDF joint and refuses to
launch if both are non-identity — but I'd rather adopt an existing pattern than invent one. Curious
how other people handle it, particularly on robots with several sensors that each ship their own
extrinsic config.

---

### Notes before you post (delete this section)

- **Title alternatives**, depending on how discoverable you want it:
  - *"Mirrored point cloud from duplicated extrinsics — a TF debugging story"* (searchable)
  - *"A valid TF tree can still be wrong: extrinsics declared twice"* (better hook)
- **Where it goes:** ROS Discourse, General category, or the Localization/Navigation subcategory
  if your instance has one.
- **Reuse this three ways** — that's the point of writing it:
  1. Your interview anchor story. It's already structured as symptom → investigation → root cause →
     generalisation, which is exactly how you should tell it out loud.
  2. The README for a TF-consistency checker, if you build the tool.
  3. A shortened LinkedIn post (cut to the fix and the three lessons, link to the Discourse thread).
- **Tone check:** the closing question is doing real work. It invites replies, which is what makes
  a Discourse thread visible, and it signals you're asking rather than lecturing. Keep it.
- **Don't** name your employer, the customer, the product, or the application. The post is stronger
  as a generic sensor-integration lesson and safer for you.
