# TODO: Structure

- Introduction + gif of running version
- Overview /Structure (see https://github.com/dietriro/rto_core) + rqtgraph
- Packages with subscribed topics, published topics, services, config (params)
- How to run / Usage (roslaunch rto_nav..)
- Install (clone, catkin build, etc.)


# NavPy

## Introduction

## Structure/Overview

## Packages

### rto_costmap_generator
#### Description
This package contains the 'costmap_generator_node', which is used for generating the local and the global costmap. 

In order to allow the use of a point representation of the mobile robot for path planning, the static map from the map server has to be padded by the radius of the mobile robot and by an additional, optional safety distance. This leads to an area in that the robot should under no circumstances operate in. Therefore, the system will efficiently prevent the robot from crashing into obstacles that are positioned in the global costmap. This area is called the area of 'hard padding'. To additionally asign a higher cost to grid cells that are close to obstacles, a so called 'soft padding' area gets generated by the costmap generator. The robot is allowed to operate in the area but path planning through parts of the 'soft padded area' will lead to an additional cost that is taken into account by the global planner. The global costmap is a map of the type OccupancyGrid. Grid cells that are occupied by obstacles have the value 100, grid cells in the area of 'hard padding' the value 99 and grid cells in the area of 'soft padding' the values from 98 to 1. Free space is represented with the vlaue 0 and unknown space with the value -1. The following figures will represent three costmaps with the same 'hard padding' area but different types and sizes of the 'soft padding' area.

<table style="margin-left: auto; margin-right: auto; table-layout: fixed; width: 100%">
  <tr>
    <td style="width: 48%;"> <img src="resources/images/map_expo02.png"></td>
    <td style="width: 48%;"> <img src="resources/images/map_expo1.png"></td>
    <td style="width: 48%;"> <img src="resources/images/map_linear1.png"></td>
  </tr>
  <tr>
    <td style="width: 48%;" valign="top"> <b>Fig.x:</b> 'Exponential' soft padding (0.2 m).
    </td>
    <td style="width: 48%;" valign="top">  <b>Fig.x:</b> 'Exponential' soft padding (1.0 m).
    </td>
    <td style="width: 48%;" valign="top">  <b>Fig.x:</b> 'Linear' soft padding (1.0 m).
    </td>
  </tr>
</table>


In a real world scenario it is not enough to make decisions based on a static global costmap, since dynamic changes in the surrounding might lead to significant changes in the global costmap. If these changes are not recognised by the system, the accuracy of the loclization will be drastically reduced. Therefore, obstacles that are not taken into account by the current version of the global costmap have to be recognized and added in order to allow a smooth and stable navigation of the mobile robot. The local costmap serves this purpose by considering the current laserscan range measurements. The figure bellow depicts the local costmap and a obstacle that is currently not part of the global costmap. 

<table style="margin-left: auto; margin-right: auto; table-layout: fixed; width: 300px;">
  <tr>
    <td style="width: 300px;"> <img src="resources/images/local_costmap.png"></td>
  </tr>
  <tr>
    <td style="width: 300px;" valign="top"> <b>Fig.x:</b> Local costmap (green).
  </tr>
</table>

#### Subscribed Topics
##### `/scan`
To receive the LaserScan messages from the hokuyo laser scanner.
##### `/odom`
To receive the Odometry messages from the odometry system.
#### Published Topics
##### `/global_costmap`
To publish the OccupancyGrid of the padded global costmap.
##### `/local_costmap`
To publish the Occupancy Grid of the local costmap for visualization purpose.
##### `/local_obstacles`
To publish the PointCloud of sensed obstacles that has been transformed to the map frame.
#### Services
##### `/switch_maps`
Service that switches the static map that gets used by the costmap generator to generate the global costmap.<br>
request: map_nr_switch [int8] <br>
response: success [bool]
##### `/clear_map`
Service that resets the global costmap to its original state that only represents the padded static map. All additionally added obstacles will be deleted.<br>
request: command [string] ('clear')<br>
response: success [bool]
##### `/add_local_map`
Service that adds elements from the local_obstacle point cloud to the global costmap.<br>
request: command [string] ('stuck')<br>
response: success [bool]
#### Configuration
`init_map_nr`: Map to start the costmap generator with.<br>
`log_times`: Log execution times of critical operations.<br>
`debug_mode`: Return debugging messages to the terminal.<br>
`global_costmap`: 
- `robot_diameter`: The diameter of the robot used for 'hard padding'.
- `safety_distance`: Additional distance used for 'hard padding'.
- `padded_val`: Value of the grid elements that are part of the 'hard padding' area.
- `apply_soft_padding`: Apply the area of 'soft padding' to the global costmap.
- `decay_distane`: Distance from the area of 'hard padding' that is affected by 'soft padding'.
- `decay_type`: Decay type of the area of 'soft padding' (linear, exponential, reciprocal).<br>

`local_costmap`:
- `length`: Width and height of the local costmap.
- `frequency`: Frequency of updating the local costmap.
- `frequency_scan`: Frequency at which the laser scanner operates.


### rto_map_server
#### Description
This package contains the rto_map_server node, which transforms a pgm file to a OccupancyGrid message and is able to store and switch between multiple maps. The map server also adds meta information that is stored in the corresponding yaml file to the the OccupancyGrid message. The pgm and yaml files should be stored in the maps folder. To create the corresponding files for a new map existing packages like the 'slam_toolbox' and the ROS navigation stack 'map_saver' can be utilized. The information of the yaml file should be added in the form of a dictionary to the rto_map_server config file.

#### Subscribed Topics
none
#### Published Topics
##### `/map`
To publish the OccupancyGrid that has been constructed based on a pgm and yaml file.
#### Services
##### `/get_map`
Service that adds elements from the local_obstacle point cloud to the global costmap.<br>
request: map_nr [int64]<br>
response: map [nav_msgs/OccupancyGrid]
#### Configuration
`debug_mode`: Return debugging messages to the terminal.<br>
`maps_nr`: The number of maps being stored on the map server.<br>
`mapx`:
- `image`: Name of the pgm image stored in the maps folder
- `resolution`: Resolution of the pixels in meter.
- `origin`: List consiting of the x, y and z coordinate of the maps origin.
- `occupied_thresh`: Threshold of the grid value for being seen as occupied.
- `free_thresh`: Threshold of the grid value for being seen as free.


### rto_local_planner
#### Description
This package contains the local_planner_node, which creates a local path and allows the robot to follow the global path to reach the navigation goal.

The local planner implemented in this package is based on the dynamic window approach, which is an online collision avoidance strategy that samples trajectories from a generated valid search space and selects the best trajectory for the current situation with the help of a cost function. The cost function consists of four different seperate costs and is minimized in order to obtain the optimal control values. The four parts of the cost function are:

- cost based on linear velocity
- cost based on the angle towards the goal
- cost based on the proximity to the global path 
- cost based on the proximity to obstacles

The overall cost for a control pair is 0 if the robot travels with its maximal linear velocity, looks directly towards the goal, is exactely on the global path and the range to the closest obstacle is as big as possible. Based on the gain factors of the different costs the local planner will exhibit a certain behaviour. If the robot, for example, should dynamically avoid obstacles that are not part of the costmap, it would make sense to reduce the gain of the cost that is based on the proximity to the global path and increase the gain of the cost that is related to the proximity to obstacles. It the robot should however follow exactelly the global path, different gain values might make more sense. The following gifs show two completely different strategies for local planning.

<table style="margin-left: auto; margin-right: auto; table-layout: fixed; width: 100%">
  <tr>
    <td style="width: 48%;"> <img src="resources/gifs/obstacle_avoidance.gif" width="350"/></td>
    <td style="width: 48%;"> <img src="resources/gifs/path_following.gif" width="350"/></td>
  </tr>
  <tr>
    <td style="width: 48%;" valign="top"> <b>Gif.x:</b> Local planner focuses on avoiding obstacles (gain values: 18 12 15 15).
    </td>
    <td style="width: 48%;" valign="top"> <b>Gif.x:</b> Local planner focuses on staying on the global path (gain values: 28 2 80 1).
    </td>
  </tr>
</table>

Of course both strategies have advantages and disadvantages and it depends on the situation which version to use.
Please be aware of the fact that the parameters are tuned for the robot to work in the gazebo simulation environment. Applying the local planner to real world conditions might require additional parameter tuning.



<table style="margin-left: auto; margin-right: auto; table-layout: fixed; width: 100%">
  <tr>
    <td style="width: 48%;"> <img src="resources/gifs/recovery1.gif" width="350"/></td>
    <td style="width: 48%;"> <img src="resources/gifs/recovery2.gif" width="350"/></td>
    <td style="width: 48%;"> <img src="resources/gifs/recovery3.gif" width="350"/></td>
  </tr>
  <tr>
    <td style="width: 48%;" valign="top"> <b>Gif.x:</b> Initializing a recovery behaviour based on a fully blocked path.
    </td>
    <td style="width: 48%;" valign="top"> <b>Gif.x:</b> Initializing a recovery behaviour based on a partly blocked path.
    </td>
    <td style="width: 48%;" valign="top"> <b>Gif.x:</b> Initializing a recovery behaviour based on a trapped robot.
    </td>
  </tr>
</table>








All other packages have been adapted from https://github.com/dietriro/rto_core and https://github.com/dietriro/rto_simulation.

## Install and how to run





