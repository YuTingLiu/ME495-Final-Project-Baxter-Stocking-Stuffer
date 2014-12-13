ME495-Final-Project-Baxter-Stocking-Stuffer
===========================================

Team members: Sabeen Admani, Chukwunyere Igbokwe, Josh Marino, and Andrew Turchina

###Table of Contents
[Objective](#Objective)  
[Equipment and Hardware Requirements](#Equipment and Hardware Requirements)  
[Preliminary Steps](#Preliminary Steps)  
[Project Overview](#Project Overview)  
[Implementation](#Implementation)  
[Dependencies](#Dependencies)  
[Main Scripts](#Main Scripts) 
[Further Improvements](#Further Improvements)  
[Conclusions](#Conclusions)  


<a name="Objective"></a> 
###Objective
The goal of this project is to get a [Baxter Robot](http://www.rethinkrobotics.com/baxter/) to act as Santa's [elf](http://en.wikipedia.org/wiki/Christmas_elf) helper.  Baxter will have a set of presents and a set of stockings in his workspace.  Baxter will identify a stocking, locate the corresponding present, and then place the present in the stocking.

Click on the image below to watch the video!
[![Screenshot](https://raw.githubusercontent.com/ChuChuIgbokwe/ME495-Final-Project-Baxter-Stocking-Stuffer/master/clippicture.png)](https://vimeo.com/114372776)


<a name="Equipment and Hardware Requirements"></a>
###Equipment and Hardware Requirements
1. Baxter Robot  
2. [ROS Indigo](http://wiki.ros.org/ROS/Installation) on Ubuntu 14.04  
3. [Asus Xtion Pro Camera](http://www.asus.com/Multimedia/Xtion_PRO/)  
4. [4 Stockings](http://www.amazon.com/Plush-Christmas-Stocking-White-Trim/dp/B001L11PQ0/ref=sr_1_9?ie=UTF8&qid=1416682045&sr=8-9&keywords=christmas+stockings)  
5. 4 Presents of different colors  
6. [4 AR tags](http://wiki.ros.org/ar_track_alvar?action=AttachFile&do=view&target=markers0to8.png)  
7. Table to place presents  
8. Wall and hangers to place stockings  

<a name="Preliminary Steps"></a>
###Preliminary Steps 
#####Setup Baxter  
Rethink Robotics has [Baxter Setup](http://sdk.rethinkrobotics.com/wiki/Baxter_Setup) instructions and [Workstation Setup](http://sdk.rethinkrobotics.com/wiki/Workstation_Setup) instructions.   
#####Setup demo area
There needs to be a table with presents within Baxter's reachable workspace along with a wall of stockings. Both QR codes and ar_tags were used in this project. QR codes can be generated online in many places. To create ar_tags, there are [generic samples](http://wiki.ros.org/ar_track_alvar?action=AttachFile&do=view&target=markers0to8.png) or one can generate by `rosrun ar_track_alvar createMarker`  

![Baxter Workspace](https://raw.githubusercontent.com/ChuChuIgbokwe/ME495-Final-Project-Baxter-Stocking-Stuffer/master/baxter_workspace_placeholder.jpeg)  


<a name="Project Overview"></a>
###Project Overview 
Outline of the steps that went into building the package that will run Baxter through the program  

1. Sweep the stockings  
2. Store the tag ID and location from each stocking  
3. Relate the stocking tag ID to a present color  
4. Sweep the table  
5. Identify the colors and locations of presents  
6. Move Baxter's gripper to the present location  
7. Pick up the present  
8. Move Baxter's gripper to the corresponding stocking location  
9. Drop the present into the stocking  


<a name="Implementation"></a>
###Implementation 

There are two different launch files in our package. ar_trackv2.launch_ is used when you simply want to run the ar_track_alvar package with your camera to see what the tag ID numbers are and test the "present identifying" when you appropriate change the ID numbers and present colors in the "needed_present_identifier.py" script to match what is specific to your project.

To run the entire sequence, you can simply do a roslaunch:
```
roslaunch baxter_stocking_stuffer visp_and_move.launch

```

Below is what the launch file used to run the stocking stuffing sequence looks like:  
```
<launch>

  <!-- Include launch file that starts AR tracker and identifies stocking -->
  <include file="$(find baxter_stocking_stuffer)/launch/ar_trackv2.launch"/>
  

  <!-- Node to open Baxter's left hand camera, after closing them all -->
  <node pkg="baxter_tools" type="camera_control.py" name="close_left_camera" output="screen"  args="-c left_hand_camera">
  </node>
  <node pkg="baxter_tools" type="camera_control.py" name="close_right_camera" output="screen"  args="-c right_hand_camera">
  </node>
  <node pkg="baxter_tools" type="camera_control.py" name="close_head_camera" output="screen"  args="-c head_camera">
  </node>

  <node pkg="baxter_tools" type="camera_control.py" name="open_left_camera" output="screen"  args="-o left_hand_camera -r 640x400">
  </node>


  <!-- Launch the tracking node -->
  <node pkg="visp_auto_tracker" type="visp_auto_tracker" name="visp_auto_tracker" output="log">
    <param name="model_path" type="string" value="$(find visp_auto_tracker)/models" />
    <param name="model_name" type="string" value="pattern" />
    <param name="debug_display" type="bool" value="True" />
        
    <remap from="/visp_auto_tracker/camera_info" to="/cameras/left_hand_camera/camera_info"/>
    <remap from="/visp_auto_tracker/image_raw" to="/cameras/left_hand_camera/image"/>
  </node>


  <!-- Node that accepts a PoseStamped() message and moves toward it, if a solution is possible. -->
  <node pkg="baxter_stocking_stuffer" type="baxtermovement.py" name="movement_node" output="screen" >
  </node>

  <!-- Once stocking has been identified, determines the pose of the stocking using QR code -->
  <node pkg="baxter_stocking_stuffer" type="poseusingidandqrcode.py" name="stocking_pose" output="screen" >
  </node> -->

  <!-- Node to publish center of object after thresholded with OpenCV -->
  <node pkg="baxter_stocking_stuffer" type="open_cv_vision.py" name="vision_node" output="screen" >
  </node>

  <!-- Node listening to center of object that publishes PoseStamped() message for Baxter's gripper to get within grapsing reach -->
  <node pkg="baxter_stocking_stuffer" type="poseusingcolordetection.py" name="color_node" output="screen" >
  </node>


  <!-- Node that moves Baxter's gripper and held present back to stocking, then releases into stocking -->
  <node pkg="baxter_stocking_stuffer" type="back_to_stocking_and_release.py" name="returns_present" output="screen" >
  </node>

</launch>
```

<a name="Dependencies"></a>
###Dependencies 
- [visp_auto_tracker](http://wiki.ros.org/visp_auto_tracker)  
- [ar _ track _ alvar](http://wiki.ros.org/ar_track_alvar)  



<a name="Main Scripts"></a>

###Main Scripts:
These nodes run in a certain sequence of steps. The way this is accomplished is by having each of these nodes listen to certain topics that contain boolean messages of true and false. Published messages of true to certain topics begin specific actions and the opposite is true for when false messages are published.

![node_map](https://raw.githubusercontent.com/ChuChuIgbokwe/ME495-Final-Project-Baxter-Stocking-Stuffer/master/node_map.png)  

<h4>needed_present_identifier.py

Overall function: This node kicks off the stocking stuffing sequence. In addition, it is needed to identify whose present Baxter needs to search for on the table. 

+ The node starts if the state of '/start/sweep' is True and Baxter moves to a set distance away from the stockings so that he can see all of the tags at once
+ When Baxter comes across an ID that is not in his "completed list" of identified presents yet, he publishes the color associated with the ID to /color_identifier and the ID to /scanned_stocking_id
+ Publishes True to /start/stockingpose in order to begin the next element in the sequence 
+ Publishes False to /state/sweep in order to end the present ID search.

Published Topics:
- `/color_identifier`
- `/scanned_stocking_id`
- `/start/sweep`
- `/start/stockingpose`
- `/baxter_movement/posestamped`

Subscribed Topics:
- `/ar_pose_marker`
- `/start/sweep`

<h4>poseusingidandqr.py

Overall function: It gets the position of the stocking, moves to it and publishes a message about its location
+ The node starts if state of  `/start/stockingpose` is True
+ It sets the state of to `pose/stocking` to False
+ It sets the state of `start/colordetection` to True at the end starting the next node

Published Topics:
- `baxter_movement/posestamped`
- `start/stockingpose`
- `pose/stocking`
- `start/colordetection`

Subscribed Topics:
- `/robot/limb/left/endpoint_state`
- `/visp_auto_tracker/object_position`
- `/ar_pose_marker`
- `scanned_stocking_id`
- `/start/stockingpose`

<h4>open_cv_vision.py

Overall function: It is used to identify a specific colored object and move it to the center of the camera's view
+ This node listens to which color object Baxter needs to find and selects from the appropriate range of HSV filter valus that it needs to use in order to properly identify the object
+ Finds the pose of the center of the object and publishes it to the /opencv/center_of_object topic

Published Topics:
- `/opencv/center_of_object`

Subscribers:
- `/cameras/left_hand_camera/image`
- `/color_identifier`


<h4>poseusingcolordetection.py

Overall function: It locates the object and moves to a position above the centre of the object and publishes it. It also moves the end effector to the object and grasps it.It then moves to the position above the table it started looking for the present from.
+ The node starts when the state of `/start/colordetection` is True. It publishes False to this topic at the end when the action is complete
+ It changes the state of `start/backtostocking` to True to start the next node

Publishers:
- `baxter_movement/posestamped`
- `start/colordetection`
- `start/backtostocking`

Subscribers:
- `/robot/limb/left/endpoint_state`
- `/opencv/center_of_object`
- `/start/colordetection`

<h4>back_to_stocking_and_release.py

Overall Function: It brings the present back to the stocking pose it found in the beginning of the sequence and opens the gripper, dropping the present into the stocking.
+ The node starts when the state of `/start/backtostocking` is True. It publishes to False when the action is complete
+ Gets the pose of the stocking that was obtained in the second step of the sequence by listening to the topic `/pose/stocking` and publishes that poseStamped message to `/baxter_movement/posestamped` to move his end effector to that pose
+ Once his arm reaches that pose, the node changes the state of /start/releasepresent to True
+ After releasing the stocking, the node changes the state of /start/backtostocking to False and /start/sweep back to True to begin the sequence again


Published Topics:
- `/baxter_movement/posestamed`
- `/start/backtostocking`
- `/start/releasepresent`
- `/start/sweep`
- `/pose/stocking`

Subscribed Topics:
- `/pose/stocking`
- `/start/backtostocking`
- `/start/releasepresent`

<a name="Further Improvements"></a>
###Further Improvements
An improvement to the project would be to get Baxter to Identify Presents and Stockings using Microsoft's Kinect or the Asus Xtion Pro Live and Point Cloud Libraries thus eliminating the need for tags. One reason for this is that point cloud libraries are more accurate than tags but are harder to get working; if we get it working, we could have it so we have pictures of each person on the stocking and have Baxter recognize who it is and then sort presents based on that. Furthermore, we could eliminate the need for color recognition of objects and locate presents based on their shape instead. Another Improvement would be to use both arms at the same time. For example, one arm could open the stocking while the other hand drops the present in, eliminating the need for having a cylinder in the stocking to hold it open.

<a name="Conclusions"></a>
###Conclusions 
It was a worthwhile project. Hopefully with some improvements we'll settle for  running Santa's shop floor since Baxter will never be able to fit down chimneys.
Ho-Ho-Ho!


