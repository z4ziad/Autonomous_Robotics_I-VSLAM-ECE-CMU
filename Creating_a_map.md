This file provides instructions for creating a 3D map using Isaac ROS VSLAM for the Autonomous Robotics I in ECE at CMU. 
## Assumption
You already installed and tested Isaac ROS VSLAM and visualized its output remotely for [Assignment 9, Part I.](https://github.com/z4ziad/Autonomous_Robotics_I-VSLAM-ECE-CMU)
## Installing OLED IP Driver
Follow the instructions I created on [this GitHub repo](https://github.com/z4ziad/Jetson-IP-on-Waveshare-OLED) to install the driver. Reboot or restart the Jetson, and its Wi-Fi IP should be displayed on the OLED screen of the Waveshare base robot.
## Installing the Gamepad driver
Raise the robot on a block so its wheels are not touching the ground. Follow the instructions I created on [this GitHub repo](https://github.com/z4ziad/Jetson-TeleOp-with-Waveshare-base-robot/tree/main) to move the robot around with the Gamepad controller provided for the course. Please see the Assignment Instructions on Canvas on how to borrow a gamepad.
Move the left joystick around to make sure the wheels move accordingly. 
## Testing Tele-Op
Shut down the Jetson, untether the robot from the wall power and display, and restart it. Now that the OLED IP driver is installed, it should display its IP on the OLED screen. Establish a remote SSH connection to it from your laptop or desktop and start the tele_op.py script (see above). Place the robot on the ground and move it around a bit to get used to controlling it with the joystick.   

### Establishing a Remote Connection with VSCode
Another option to connect remotely the Jetson is through VScode running on your laptop or desktop.
* Open VScode and install remote SSH Extension.
* Connect the Jetson IP and open directory. You can now edit and save files remotely.
* You can also get SSH terminals to run VSLAM remotely.  
**Note** When you connect via VSCode, a VSCode server is installed on the Jetson which can take up some memory. If you ever run out of memory, you can stop VSCode and restart the Jetson to free up that memory. You can then rely solely on SSH connections.  

## Running VSLAM and Creating the 3D Map
Launch VSLAM and visualize its output using Foxglove as you did in Part I of this Assignment
Create a 3D map of the robot enclosure made for the course. See instructions on Canvas on when and where the course robotic enclosure will be available.   
## Saving the Map
Once you have a map that you would like to save for later localization, you can save to a file on secondary storage like the SSD. While Isaac ROS VSLAM is running on the Jetson, establish a new remote SSH connection to the Jetson and connect the Isaac_ros-dev-aarch64 container:
```bash
docker exec -it -u admin isaac_ros_dev-aarch64-container /bin/bash 
```
Save the current map to file:
```bash
# 1. Verify the service is available
ros2 service list | grep visual_slam

# 2. Confirm the type
ros2 service type /visual_slam/save_map

# 3. Check the exact fields
ros2 interface show isaac_ros_visual_slam_interfaces/srv/FilePath

# 4. Create your target directory
mkdir -p /workspaces/isaac_ros-dev/maps

# 5. Save the map
ros2 service call /visual_slam/save_map isaac_ros_visual_slam_interfaces/srv/FilePath "{file_path: '/workspaces/isaac_ros-dev/maps/my_map'}"
```
## Loading the Map and Localizing the Robot
### The Concept
Now suppose you already have a map you created and saved with cuVSLAM, and you want to start VSLAM with that map. cuVSLAM, however, does not search the entire map to localize the robot. It can only localize with a hint about the current robot pose within the map. Localizing the robot anywhere on a map is better handled by cuVGL, Nvidia's Visual Global Localization. cuVGL can estimate a robot's global pose using its database of keyframe images and a Bag-of-Words index. cuVGL first estimates a global pose to bootstrap cuVSLAM localization. Once cuVSLAM successfully localizes, cuVGL stops, and cuVSLAM begins continuously localizing in the map frame.   

### Procedure Overview
Since we are not using cuVGL in this assignment, we can ask cuVSLAM to localize the robot using a hint. The default hint location is at the map origin, {x:0, y:0, z:0}, with a quaternion orientation of {x:0, y:0, z:0, w:1}, which represents no rotation. Therefore, if you place the robot roughly in the same spot, within about a 1.5-meter radius, and with the same orientation (i.e., the default hint) as when you started VSLAM and created the map, then cuVSLAM should be able to successfully load the saved map and localize within it.   
### Localize at Startup
It is easier to test cuVSLAM localization in an existing map if we do it at startup. Therefore, we need to pass arguments to the cuVSLAM node at startup to tell it to load our saved map and then localize within it.   
### Passing Arguments Through the Launch File
To pass command-line arguments to a node started with a ROS2 "launch" file, the launch file must receive the arguments and pass them to the node. Here is how to do it:     

Save the VSLAM launch file before you modify it:   
   ```bash
   cd /opt/ros/humble/share/isaac_ros_visual_slam/launch/
   cp isaac_ros_visual_slam_realsense.launch.py isaac_ros_visual_slam_realsense_orig.launch.py
   ```
Sudo edit the `isaac_ros_visual_slam_realsense.launch.py` to make the following changes:
  * Add the needed Python modules for launch arguments and configurations:
  ```Python
  import launch
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import ComposableNodeContainer, Node
from launch_ros.descriptions import ComposableNode, ParameterValue
```  

   * Declare launch arguments first thing inside the `generate_launch_description()` function:
```Python
def generate_launch_description():

    # Declare launch arguments
    localize_on_startup_arg = DeclareLaunchArgument(
        'localize_on_startup',
        default_value='False',
        description='Whether to localize against an existing map on startup.',
    )
    load_map_folder_path_arg = DeclareLaunchArgument(
        'load_map_folder_path',
        default_value='',
        description='Path to the map folder to load at startup.',
    )
    ...
    ...
```
  * Enable the localization and mapping option by **adding** the following parameter to the `visual_slam_node`
```Python
'enable_localization_n_mapping': True
```
  * Add the parameters arguments `localize_on_startup` and `load_map_folder_path` to the list of parameters for the `visual_slam_node`:
  ```Python
            'localize_on_startup': ParameterValue(
                LaunchConfiguration('localize_on_startup'),
                value_type=bool,
            ),
            'load_map_folder_path': ParameterValue(
                LaunchConfiguration('load_map_folder_path'),
                value_type=str,
            ),
```  
  * Finally, make sure the `generate_launch_description()` functions passes the arguments to the node upon returning:
```Python
return launch.LaunchDescription([
        localize_on_startup_arg,
        load_map_folder_path_arg,
        visual_slam_launch_container,
        realsense_camera_node])
```
Now that you modified the VSLAM launch file to pass the optional arguments the load a map and localize in it, let's test it out.   
   * Position the robot at the map origin marked in your map when you saved it. This should be the location of when started creating the map with VSLAM. Also, orient the robot roughly in the same direction as you did when you started creating the map.
   * Launch VSLAM with the arguments to load the map and localize:
  ```bash
  cd \workspaces\isaac_ros-dev\
  ros2 launch isaac_ros_visual_slam isaac_ros_visual_slam_realsense.launch.py localize_on_startup:=True load_map_folder_path:=/workspaces/isaac_ros-dev/<path_to_your_map_dir>   
```
  * Make sure to supply the proper path above where you saved your map.    
  
VSLAM should launch and if it is able to load the map and localize, you should see the following VSLAM output message scrolling by:
```text
[visual_slam_node]: Successfully localized at {0.044609, 0.003771, -0.003902}
```
Note that {x, y, z} values above are where the VSLAM thinks the robot is currently after localization.    

Reconnect to Foxglove and check that the 3D map is loaded and that the robot visually looks where it is currently. It might take a few seconds to update the map. You can continue building the map and saving it as a new one if you would like.

