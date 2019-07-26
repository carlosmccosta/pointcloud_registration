# pointcloud_registration

Configuration for the [drl](https://github.com/carlosmccosta/dynamic_robot_localization) perception pipeline for performing 6 DoF point cloud registration for correcting small offsets between a reference point cloud in a calibrated frame and a point cloud captured from a 3D sensor.

Check the configurations of the repository [object_recognition](https://github.com/carlosmccosta/object_recognition)
and the documentation of the [dynamic_robot_localization](https://github.com/carlosmccosta/dynamic_robot_localization) ROS package for customizing the perception pipeline for your specific use case.



## Calibration of the coordinate systems for object segmentation

For segmenting the target object from the environment, a region of interest is used (defined in [yaml/filters_ros.yaml](yaml/filters_roi.yaml)) that is applied to the point cloud after transforming it to a calibrated reference frame (frame_id: table).

The [charuco_detector](https://github.com/carlosmccosta/charuco_detector) can be used for recalibrating the charuco frame, followed by the update of the TF publisher in [launch/config/tfs_user_coordinate_systems.launch](launch/config/tfs_user_coordinate_systems.launch) with the data given by the charuco_detector pose topic.

Then, the table frame can be adjusted in relation to the charuco frame.



## Creation of the reference point clouds database

For saving the reference point clouds to disk after being segmented, filtered and transformed to the base_link frame_id, run:
```
roslaunch pointcloud_registration bringup.launch use_cloud_storage_mode:=true
```

Then, place the target object on top of the table, trigger the 3D sensor and send a goal to the perception pipeline with the file name that will be given to the reference point cloud.

The name can be an absolute path or a relative path in relation to the [reference_pointclouds_database_folder_path] specified in [launch/config/pointcloud_registration.launch](launch/config/pointcloud_registration.launch).

The pipeline was also configured with an euclidean clustering algorithm for picking the largest cluster from the environment (0), or a given cluster specified in the perception pipeline goal (clusters are sorted by decreasing size by default).

Example of goal for triggering the saving of a reference point cloud with surface normals information:
```
rostopic pub /pointcloud_storage/ObjectRecognitionSkill/goal object_recognition_skill_msgs/ObjectRecognitionSkillActionGoal "header:
  seq: 0
  stamp:
    secs: 0
    nsecs: 0
  frame_id: ''
goal_id:
  stamp:
    secs: 0
    nsecs: 0
  id: ''
goal:
  objectModel: 'reference_point_cloud_1'
  clusterIndex: 0"
```


## Point cloud registration usage

For starting the point cloud registration pipeline, run:
```
roslaunch pointcloud_registration bringup.launch
```

For triggering the point cloud registration, send a goal to the perception pipeline, specifying the reference point cloud filename (absolute path or relative to the [reference_pointclouds_database_folder_path]) and the cluster index to use (0 -> largest cluster).

Example of goal for triggering the point cloud registration:
```
rostopic pub /pointcloud_registration/ObjectRecognitionSkill/goal object_recognition_skill_msgs/ObjectRecognitionSkillActionGoal "header:
  seq: 0
  stamp:
    secs: 0
    nsecs: 0
  frame_id: ''
goal_id:
  stamp:
    secs: 0
    nsecs: 0
  id: ''
goal:
  objectModel: 'reference_point_cloud_1'
  clusterIndex: 0"
```

The point cloud registration result is published in a ROS [geometry_msgs/PoseStamped](http://docs.ros.org/api/geometry_msgs/html/msg/PoseStamped.html) topic, and can be inspected using:
```
rostopic echo /pointcloud_registration/localization_pose
```
or
```
rosrun tf tf_echo base_link registration_correction
```


## Visual inspection of point cloud registration

For monitoring the registration state, run:
```
roslaunch pointcloud_registration rviz.launch
```


## Notes

If your 3D sensor does not provide surface normals in the point cloud, change to false the argument [sensor_provides_surface_normals] in [launch/config/pointcloud_registration.launch](launch/config/pointcloud_registration.launch)
