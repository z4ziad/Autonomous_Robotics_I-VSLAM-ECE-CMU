# VSLAM for Autonomous Robotics I, ECE, CMU
This repo explains how to start Isaac ROS VSLAM in Docker 
after installing and running Isaac ROS Object Detection and Object 
Tracking on Assignment 8 in Autonomous Robotics I.

## Assumptions
You completed Assingmnet 8 with Object Tracking with YOLOv8 in Docker on Jetson Orin Nano and the RealSense 435i camera. 
## Install the Required Assets
1. Start the `isaac_ros_dev-aarch64-container`    
```shell
docker start -a -i isaac_ros_dev-aarch64-container```
```
2. Inside the container, make sure we have the required libraries
```shell
sudo apt-get install -y curl jq tar
```
3. Then download the assets from NGC:
```shell
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
In this step we install VSLAM from binaries since we don't intend to modify the souce code.   
Inside the container:   
```shell
sudo apt-get update
```
then...
```shell
sudo apt-get install -y ros-humble-isaac-ros-visual-slam
```
## Modify the RealSense Infrared Camera Frame Rate
We need to modify the default frame rate configuration for the RealSense camera so that it runs with at least at 30 Hz or 33 ms per frame. The `visual_vslam_node` expects the delta time between frames to be less than or equal to 34 ms.
```shell
cd /opt/ros/humble/share/isaac_ros_realsense/config
```
We need to edit the `realsense_stereo.yaml` file, but let's copy it first just in case:
```shell
sudo cp realsense_stereo.yaml realsense_stereo.yaml_orig
```
Now with `sudo` priviledges, edit the `realsense_stereo.yaml` file to change the profile as follows:
```yaml
profile: '640x360x30'
```
The profile with `'640x360x90'` with 90 FPS is not valid.    

## Launch VSLAM
Now that your installation is complete, let's launch VSLAM. We are assuming that you already installed `ros_humble-isaac-ros-example` and `ros-humble-isaac-ros-realsense` for the previous Assinments. If you get a message saying that the "realsense camera was not found", unplug and then replug the camera cable (There is software fix for this issue, but let's not worry about it for the time being).    
```shell
cd /workspaces/isaac_ros_dev
```
```shell
ros2 launch isaac_ros_examples isaac_ros_examples.launch.py launch_fragments:=realsense_stereo_rect,visual_slam \
interface_specs_file:=${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_visual_slam/quickstart_interface_specs.json \
base_frame:=camera_link camera_optical_frames:="['camera_infra1_optical_frame', 'camera_infra2_optical_frame']"
```
## Visualize the Output
Connect the running Docker container:
```shell
docker exec -it -u admin isaac_ros_dev-aarch64-container /bin/bash
```
Then run RViz to visualize the vslam output:
```shell
rviz2 -d $(ros2 pkg prefix isaac_ros_visual_slam --share)/rviz/default.cfg.rviz
```




