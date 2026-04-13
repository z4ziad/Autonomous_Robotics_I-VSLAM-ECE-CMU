# Setting up VSLAM for Autonomous Robotics I, ECE, CMU
This repo explains how to set up Isaac ROS VSLAM in Docker 
after installing and running Isaac ROS Object Detection and Object Tracking on Assignment 8 in Autonomous Robotics I. This is Part 1 of the VSLAM. In part 2, we move the robots with a gamepad controller to create a map of the robot's environment and test its localization ability. 

## Assumptions
You completed Assignment 8 with Object Tracking with YOLOv8 in Docker on Jetson Orin Nano and the RealSense 435i camera. 
## Install the Required VSLAM Assets
1. Start the `isaac_ros_dev-aarch64-container`    
```shell
docker start -a -i isaac_ros_dev-aarch64-container```
```
2. Inside the container, make sure we have the required libraries
```shell
sudo apt-get install -y curl jq tar
```
3. Then download the assets from NGC by running these commands:
```bash
NGC_ORG="nvidia"
NGC_TEAM="isaac"
PACKAGE_NAME="isaac_ros_visual_slam"
NGC_RESOURCE="isaac_ros_visual_slam_assets"
NGC_FILENAME="quickstart.tar.gz"
MAJOR_VERSION=3
MINOR_VERSION=2
VERSION_REQ_URL="https://catalog.ngc.nvidia.com/api/resources/versions?orgName=$NGC_ORG&teamName=$NGC_TEAM&name=$NGC_RESOURCE&isPublic=true&pageNumber=0&pageSize=100&sortOrder=CREATED_DATE_DESC"
AVAILABLE_VERSIONS=$(curl -s \
    -H "Accept: application/json" "$VERSION_REQ_URL")
LATEST_VERSION_ID=$(echo $AVAILABLE_VERSIONS | jq -r "
    .recipeVersions[]
    | .versionId as \$v
    | \$v | select(test(\"^\\\\d+\\\\.\\\\d+\\\\.\\\\d+$\"))
    | split(\".\") | {major: .[0]|tonumber, minor: .[1]|tonumber, patch: .[2]|tonumber}
    | select(.major == $MAJOR_VERSION and .minor <= $MINOR_VERSION)
    | \$v
    " | sort -V | tail -n 1
)
if [ -z "$LATEST_VERSION_ID" ]; then
    echo "No corresponding version found for Isaac ROS $MAJOR_VERSION.$MINOR_VERSION"
    echo "Found versions:"
    echo $AVAILABLE_VERSIONS | jq -r '.recipeVersions[].versionId'
else
    mkdir -p ${ISAAC_ROS_WS}/isaac_ros_assets && \
    FILE_REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/\
versions/$LATEST_VERSION_ID/files/$NGC_FILENAME" && \
    curl -LO --request GET "${FILE_REQ_URL}" && \
    tar -xf ${NGC_FILENAME} -C ${ISAAC_ROS_WS}/isaac_ros_assets && \
    rm ${NGC_FILENAME}
fi
```
## Install Isaac ROS VSLAM
In this step we install VSLAM from binaries since we don't intend to modify the source code.   
**Inside the Docker container:**   
```shell
sudo apt-get update 
```
then...
```shell
sudo apt-get install -y ros-humble-isaac-ros-visual-slam
```
## Modify the RealSense Infrared Camera Frame Rate
Since this VLSAM package relies on the stereo vision of the infrared cameras on the RealSense 435i, we need to modify the default infrared frame rate configuration so we get a "good" balance between the frame rate and resolution. Running at 30Hz should be good enough for mapping while the robot is moving and detecting good features to track (GFTT). A profile of 640x480x30 is workable profile.    
We need also to tell `visual_vslam_node` expect 30Hz frame rate, i.e., at least 33ms, otherwise, it would think some frames are dropped.    
Run the following commands to find out the valid modes for your RealSense camera. The modes depend on the firmware version, which should be at least 5.13
```bash
rs-enumerate-devices -c
```
The above command puts out a lot of output. Scroll up and look for the "Supported modes:" section. One the valid modes should be `Infrared 1 640x480 Y8 @ 30/15/6 Hz` so let's go with that. First backup the original VSLAM launch file:
```bash
cd /opt/ros/humble/share/isaac_ros_visual_slam/launch
cp isaac_ros_visual_slam_realsense.launch.py isaac_ros_visual_slam_realsense.launch_orig.py
```
Now sudo edit the file `isaac_ros_visual_slam_realsense.launch.py` to change the following parameters:
```python
'depth_module.profile': '640x480x30'    
'image_jitter_threshold_ms': 44.00
```   
## Launch VSLAM  
Make sure to plug the RealSense camera. Then launch VSLAM
```shell
cd /workspaces/isaac_ros_dev
ros2 launch isaac_ros_visual_slam isaac_ros_visual_slam_realsense.launch.py
```
If you get a message saying the RealSense device not found, please unplug and re-plug the camera so the Docker container can see it.   

Check that the VSLAM topics are being published by the `visual_slam` node.
Connect to the running Docker container:
```shell
docker exec -it -u admin isaac_ros_dev-aarch64-container /bin/bash
```
Inside the Docker container:
```shell
ros2 topic list
```
and you should see the topics:
```
/camera/accel/imu_info
/camera/accel/metadata
/camera/accel/sample
/camera/extrinsics/depth_to_accel
/camera/extrinsics/depth_to_gyro
/camera/extrinsics/depth_to_infra1
/camera/extrinsics/depth_to_infra2
/camera/gyro/imu_info
/camera/gyro/metadata
/camera/gyro/sample
/camera/imu
/camera/infra1/camera_info
/camera/infra1/image_rect_raw
/camera/infra1/image_rect_raw/compressed
/camera/infra1/image_rect_raw/compressedDepth
/camera/infra1/image_rect_raw/theora
/camera/infra1/metadata
/camera/infra2/camera_info
/camera/infra2/image_rect_raw
/camera/infra2/image_rect_raw/compressed
/camera/infra2/image_rect_raw/compressedDepth
/camera/infra2/image_rect_raw/theora
/camera/infra2/metadata
/parameter_events
/rosout
/tf
/tf_static
/visual_slam/imu
/visual_slam/status
/visual_slam/tracking/odometry
/visual_slam/tracking/slam_path
/visual_slam/tracking/vo_path
/visual_slam/tracking/vo_pose
/visual_slam/tracking/vo_pose_covariance
/visual_slam/vis/gravity
/visual_slam/vis/landmarks_cloud
/visual_slam/vis/localizer
/visual_slam/vis/localizer_loop_closure_cloud
/visual_slam/vis/localizer_map_cloud
/visual_slam/vis/localizer_observations_cloud
/visual_slam/vis/loop_closure_cloud
/visual_slam/vis/observations_cloud
/visual_slam/vis/pose_graph_edges
/visual_slam/vis/pose_graph_edges2
/visual_slam/vis/pose_graph_nodes
/visual_slam/vis/velocity 
```
If all is good, then let's visualize the VSLAM output.
## Visualize VSLAM Output with Foxglove
It is critical in robotics not to visualize the output on the same compute platform as the robot as it might take away too much compute resources from the robot itself. [Foxglove](https://foxglove.dev/) is a service that lets us easily visualize the output for VSLAM on your laptop or desktop.  

**Note 1:** Sign up for Foxglove using your CMU email so you can get the educational version free with most bandwidth and storage.

**Note 2:** If you are using a browser to visualize your data, you must use Chrome browser to be compatible with Foxglove background scripts and must give it permission to load scripts. 

At first, we want to test Foxglove while the robot being **tethered to power and display and being static**. I will have separate document for testing visualization with the robot being **untethered and mobile.**  
     
To get started, connect to the running `isaac_ros_dev-aarch64-container` container:    
```bash
docker exec -it -u admin isaac_ros_dev-aarch64-container /bin/bash
```
Then follow the [instructions](https://docs.foxglove.dev/docs/getting-started/frameworks/ros2?modality=live) from Foxglove ROS2 to install the **foxglove bridge** inside the `isaac_ros_dev-aarch64-container`. Launch the foxglove bridge by connecting your laptop/desktop to it to visualize the topics' data.      

You can download and apply the foxglove JSON layout file from this repo to make it easier to get the right panels.  

In one of the camera panels: pick one of the two infrared stereo vision, e.g. `/camera/infra1/image_rect_raw`.

In the 3D panel, turn on `/visual_slam/vis/landmarks_cloud` so you can see the created map. Move the robot manually with your hands to verify that the visualization is working. You can also turn on the `/visual_slam/tracking/slam_path` and 
`/visual_slam/tracking/vo_path`. Here is an example screenshot of my foxglove visualization. ![](https://github.com/z4ziad/Auto_Robo_1_VSLAM/blob/main/Screenshot_chrome_1.png)    
    
Try to pan in and out in 3D map with mouse scroll. Hold down the Shift key and click-and-drag to examine the 3D map structure.   

Next part, moving the robot with a gamepad controller to create a map!






