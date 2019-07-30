# pointcloud_registration

Configuration for the [drl](https://github.com/carlosmccosta/dynamic_robot_localization) perception pipeline for performing 6 DoF point cloud registration for correcting offsets between a reference point cloud in a calibrated frame (base_link) and a point cloud captured from a 3D sensor.

This package was tested for correcting robotic arm trajectories that are taught for a given object type and then need to be corrected when new parts of the same type appear in the robot work space (with an unknown offset).

As such, even though this package is generic and can be used for other use cases, it was configured by default for the following usage:
- Capture a reference point cloud containing the object on top of the calibrated table
- Program the robot trajectories for operating on the target object (saved in a user frame, set to identity in relation to the robot base)
- Later on, when a object of the same type arrives in the robot workspace (for painting, for example):
  - Capture a new point cloud (with the point cloud publisher latched, or send the goal first and then trigger the sensor capture, because the perception pipeline only subscribes to point cloud topics during an active goal)
  - Send o goal to the action server of this package for correcting the offset (indicating the reference point cloud to use)
  - Update the user frame in the robot controller with the offset computed by the pointcloud_registration action server
  - Run the robot trajectory on the updated user frame

Check the configurations of the repository [object_recognition](https://github.com/carlosmccosta/object_recognition)
and the documentation of the [dynamic_robot_localization](https://github.com/carlosmccosta/dynamic_robot_localization) ROS package for customizing the perception pipeline for your specific use case. Namely, customizing the filtering stage for adjusting the number of points selected for the reference and sensor point clouds (depends on the object size and registration accuracy that you need) and also the registration timeouts associated with the feature matching and ICP registration.



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

This package was tested for working with large objects (around 1 m squared) and mostly with a rectangular / box shape (for example, baguette trays). As such, by default the perception system is configured for using principal component analysis (PCA) for estimating the initial alignment between the reference point cloud and the sensor point cloud. This was suitable for our use case because the full object is visible and its box shape is ideal for PCA (the centroid and PCA axis give a good initial alignment).

The default system configuration can be started by running:
  ```
  roslaunch pointcloud_registration bringup.launch
  ```

The system can also be configured to use feature matching instead of PCA, by running:
  ```
  roslaunch pointcloud_registration bringup.launch use_feature_matching_for_initial_alignment:=true use_pca_for_initial_alignment:=false
  ```

Alternatively, if you expect very small offsets between the reference point cloud and the sensor point cloud, you can disable the initial alignment stage (set feature matching and pca arguments above to false) and rely only on ICP for performing the point cloud registration.


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

The point cloud registration result is published in a ROS [geometry_msgs/PoseStamped](http://docs.ros.org/api/geometry_msgs/html/msg/PoseStamped.html) topic and also in TF. It can be inspected using:
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

- This package was tested with a PhotoNeo PhoXi 3D Scanner XL, which is a very accurate 3D sensor that provides surface normals in the point cloud published by its ROS driver. If your 3D sensor does not provide surface normals in the point cloud, you need to change to false the argument [sensor_provides_surface_normals] in [launch/config/pointcloud_registration.launch](launch/config/pointcloud_registration.launch) (in order to add to the perception pipeline a normal estimation module).
