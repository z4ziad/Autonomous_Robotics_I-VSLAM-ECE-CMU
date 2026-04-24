This repo provides instructions for creating a 3D map using Isaac ROS VSLAM for the Autonomous Robotics I in ECE at CMU. 
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
While Isaac ROS VSLAM is running on the Jetson, establish a new remote SSH connection to the Jetson and connect the Isaac_ros-dev-aarch64 container:
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
Stop Isaac ROS VSLAM and any Foxglove visualization. Move the robot to a new location in the enclosure that was mapped (this is called the kidnapped robot problem).    
Restart VSLAM and the Foxglove visualization. Run the following commands and inspect their output: 
```bash
ros2 service type /visual_slam/localize_in_map
ros2 interface show isaac_ros_visual_slam_interfaces/srv/LocalizeInMap
```  
Load the map and localize:
```bash
ros2 service call /visual_slam/localize_in_map isaac_ros_visual_slam_interfaces/srv/LocalizeInMap "
map_folder_path: '/path/to/your/map'
pose_hint:
  position:
    x: 0.0
    y: 0.0
    z: 0.0
  orientation:
    x: 0.0
    y: 0.0
    z: 0.0
    w: 1.0"
```
Watch on Foxglove if the robot is able to localize itself on the new location in the map.   
* The pose_hint doesn't need to be exact, but it should be reasonably close to the robot's actual position in the map.
* If localization is successful, cuVSLAM will load the map into memory. Otherwise, it will continue building a new map. 
