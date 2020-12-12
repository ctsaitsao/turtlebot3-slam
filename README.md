# ME495 (Embedded Systems for Robotics) Homework 4

Author: Christopher Tsai

## Overview

This package contains code for various mapping and navigation tasks on the TurtleBot3 Burger, such as manual navigation SLAM for map generation, localization and navigation from a pre-loaded map, SLAM and path planning using a pose goal, and autonomous frontier-based exploration.

## Dependencies

- turtlebot3
- slam_toolbox

## Nodes and Launchfiles

- `start_slam.launch`: used to map the environment manually by controlling the turtlebot using arrow keys through the `turtlebot3_teleop_key` node.
- `nav_stack.launch`: used to navigate a pre-mapped environment using path planning and pose goals.
- `slam_stack.launch`: used to simultaneously map the environment and navigate using path planning and pose goals.
- `exploration.launch`: used to conduct autonomous exploration of an environment using a frontier-based approach and SLAM.
- `exploration` node: node that carries out [frontier exploration algorithm](#frontier-exploration-algorithm).

## Configuration Options

Code can be run in Gazebo instead of in real life. To do this, add the option `launch_gazebo:=true` to any of the launchfiles.

## Usage Instructions

1. Create a new workspace and clone the demonstration code.
    ```
    # Create a new workspace
    mkdir -p ws/src

    # clone the demonstration code
    cd ws/src
    git clone https://github.com/ME495-EmbeddedSystems/homework-3-ctsaitsao.git homework4

    # return to ws root
    cd ../
    ```

2. Build the workspace and activate it.
    ```
    catkin_make install
    . devel/setup.bash
    ```

3. Run the launchfile of choice, for example:
    ```
    roslaunch homework4 start_slam.launch
    ```

4. SSH into the turtlebot and start its base communication.
    ```
    ssh ubuntu@turtlebot.local
    roslaunch turtlebot3_bringup turtlebot3_robot.launch
    ```

### Time Synchronization Issues

I experienced some time synchronization issues when running the code on a real-life turtlebot. The time on the turtlebot and the time on my computer were not the same. A couple fixes for this are:
- When SSH'd into the turtlebot, run:
    ```
    sudo date -s @`(date -u +"%s" )`
    ```
- If the above fix doesn't work, connect turtlebot to router through ethernet cable and run:
    ```
    timedatectl set-ntp true
    ```
  Reboot and check that timedatectl returns `System clock synchronized: yes`.

## Demos

- [`start_slam.launch`](https://youtu.be/UYFy0_s_GdQ)
- [`nav_stack.launch`](https://youtu.be/OLLRxEmMZLc)
- [`slam_stack.launch`](https://youtu.be/qAmRSt5EXVg) 
- [`exploration.launch`](https://youtu.be/KCE35dVK1f8) 

## Frontier Exploration Algorithm

1. Get map data from `/map` topic. This data contains a row-major 1D list of map values from `slam_toolbox`, where a value of 100 corresponds to an occupied cell, a value of 0 corresponds to an unoccupied cell, and a value of -1 corresponds to an unknown cell. We are trying to find the "frontier" cells that lie in between unoccupied and unknown cells.
2. Transform 1D list of map values into a 2D np.ndarray.
3. From `/map`, get pose of lower left corner of map in terms of `map` frame (a standard frame of the ROS navigation stack). Broadcast this pose as a transform from `map` to `lower_left_corner` using `tf2_ros`.
4. Get image gradient of map array. I used my own gradient function that uses a Roberts Cross convolutional kernel but any gradient/edge detection algorithm works, such as Sobel and Canny.
5. Get global costmap data from `/move_base/global_costmap/costmap` topic.
6. Transform 1D list of global costmap values into a 2D np.ndarray.
7. From image gradient, find the cell positions where the gradient is above some threshold (in this case 150) and the global costmap is below some threshold (in this case 50). This is basically selecting frontier cells that are not near obstacles or walls.
8. Pick one of the frontier cells at random.
9. Use `tf2_ros` to send transform of of the position of this frontier cell (in a new `goal` frame) in the `lower_left_corner` frame, accounting for meters/cell resolution.
10. From TF server, acquire transform from `map` to `goal`.
11. Send this transform's x and y positions to `move_base` and wait for robot to complete its trajectory.
12. Loop back to first step.
