## Project: Perception Pick & Place
In this project,  we have the PR2 robot that is outfitted with an RGB-D sensor. This sensor however is a bit noisy, much like real sensors.

Givena cluttered tabletop scenario, a perception pipeline must be implemented  to identify target objects from a so-called “Pick-List” in that particular order, pick up those objects and place them in corresponding dropboxes.

The image below shows the setup for test world 1.

![demo-1](https://user-images.githubusercontent.com/20687560/28748231-46b5b912-7467-11e7-8778-3095172b7b19.png)



In following sections, each part of the pipeline is explained.


[//]: # (Image References)

[image1]: ./writeup_images/no_filter.png
[image2]: ./writeup_images/filtered.png
[image3]: ./writeup_images/cluster.png
[image4]: ./writeup_images/conf_mat_1.png
[image5]: ./writeup_images/conf_mat_2.png
[image6]: ./writeup_images/conf_mat_3.png
[image7]: ./writeup_images/obj_det_1.png
[image8]: ./writeup_images/obj_det_2.png
[image9]: ./writeup_images/obj_det_3.png

---
### Filtering and RANSAC plane fitting
The first step was to apply statistical outlier filtering to remove noise and have a cleaner point cloud by using a statistical outlier filer object from the PCL python library. The parameter values used for this part are summarized below:

<table class="tg">
  <tr>
    <th class="tg-us36">Filter<br></th>
    <th class="tg-us36">Parameter<br></th>
    <th class="tg-us36">Value</th>
  </tr>
  <tr>
    <td class="tg-us36" rowspan="2">Statistical Outlier<br></td>
    <td class="tg-us36">mean_k</td>
    <td class="tg-us36">15</td>
  </tr>
  <tr>
    <td class="tg-us36">std_dev_mul_thresh</td>
    <td class="tg-us36">0.1<br></td>
  </tr>
  <tr>
    <td class="tg-us36">Voxel Grid Downsampling</td>
    <td class="tg-us36">LEAF_SIZE</td>
    <td class="tg-us36">0.01</td>
  </tr>
  <tr>
    <td class="tg-yw4l" rowspan="2">PassThrough Filter<br></td>
    <td class="tg-yw4l">axis_min</td>
    <td class="tg-yw4l">0.6</td>
  </tr>
  <tr>
    <td class="tg-yw4l">axis_max</td>
    <td class="tg-yw4l">0.77<br></td>
  </tr>
</table>

Below are two images that show the raw noisy cloud, and the filtered objects cloud

![alt text][image1]
![alt text][image2]

### Clustering for segmentation implemented
To perfrom clustering, a PCL library function called EuclideanClusterExtraction() was used to perform a DBSCAN cluster search on your 3D point cloud. The parameters used for Euclidean Clustering are given in the table below:

<table class="tg">
  <tr>
    <th class="tg-us36">Parameter<br></th>
    <th class="tg-us36">Value</th>
  </tr>
  <tr>
    <td class="tg-us36">ClusterTolerance</td>
    <td class="tg-us36">0.05<br></td>
  </tr>
  <tr>
    <td class="tg-us36">MinClusterSize</td>
    <td class="tg-us36">100<br></td>
  </tr>
  <tr>
    <td class="tg-us36">MaxClusterSize</td>
    <td class="tg-us36">3000<br></td>
  </tr>
</table>

The output of clustering for segmentation for Test Word 1 is shown in the image below:

![alt text][image3]


#### Features extraction and SVM training

To train an SVM model for object detection, the color and normal histograms were used as the input features to the mode. The methods used to extract the color and normal histograms are shown in the code block below:

```python
def compute_color_histograms(cloud, using_hsv=False):

    # Compute histograms for the clusters
    point_colors_list = []

    # Step through each point in the point cloud
    for point in pc2.read_points(cloud, skip_nans=True):
        rgb_list = float_to_rgb(point[3])
        if using_hsv:
            point_colors_list.append(rgb_to_hsv(rgb_list) * 255)
        else:
            point_colors_list.append(rgb_list)

    # Populate lists with color values
    channel_1_vals = []
    channel_2_vals = []
    channel_3_vals = []

    for color in point_colors_list:
        channel_1_vals.append(color[0])
        channel_2_vals.append(color[1])
        channel_3_vals.append(color[2])
    
    # TODO: Compute histograms
    channel_1_hist = np.histogram(channel_1_vals, bins=nbins, range=bins_range)
    channel_2_hist = np.histogram(channel_2_vals, bins=nbins, range=bins_range)
    channel_3_hist = np.histogram(channel_3_vals, bins=nbins, range=bins_range)

    # TODO: Concatenate and normalize the histograms
    # Concatenate the histograms into a single feature vector
    hist_features = np.concatenate((channel_1_hist[0], channel_2_hist[0], channel_3_hist[0])).astype(np.float64)
    # Normalize the result
    normed_features = hist_features / np.sum(hist_features)

    return normed_features 


def compute_normal_histograms(normal_cloud):
    norm_x_vals = []
    norm_y_vals = []
    norm_z_vals = []

    for norm_component in pc2.read_points(normal_cloud,
                                          field_names = ('normal_x', 'normal_y', 'normal_z'),
                                          skip_nans=True):
        norm_x_vals.append(norm_component[0])
        norm_y_vals.append(norm_component[1])
        norm_z_vals.append(norm_component[2])

    # TODO: Compute histograms of normal values (just like with color)
    norm_x_hist = np.histogram(norm_x_vals, bins=nbins, range=bins_range)
    norm_y_hist = np.histogram(norm_y_vals, bins=nbins, range=bins_range)
    norm_z_hist = np.histogram(norm_z_vals, bins=nbins, range=bins_range)

    # TODO: Concatenate and normalize the histograms
    hist_features = np.concatenate((norm_x_hist[0], norm_y_hist[0], norm_z_hist[0])).astype(np.float64)
    # Normalize the result
    normed_features = hist_features / np.sum(hist_features)
    
    return normed_features

```
The parameters used in feature extraction are stated in the table below:

<table class="tg">
  <tr>
    <th class="tg-us36">Parameter<br></th>
    <th class="tg-us36">Value</th>
  </tr>
  <tr>
    <td class="tg-us36">Number of bins</td>
    <td class="tg-us36">32<br></td>
  </tr>
  <tr>
    <td class="tg-us36">Color Bin Range</td>
    <td class="tg-us36">[0,255]<br></td>
  </tr>
    <tr>
    <td class="tg-us36">Normal Bin Range</td>
    <td class="tg-us36">[-1,1]<br></td>
  </tr>
  <tr>
    <td class="tg-us36">Using_HSV</td>
    <td class="tg-us36">True<br></td>
  </tr>
</table>

To make the model robust, the number of training instances had to be large enough to provide acceptable detection accuracy. The training instances are then fed to the SVM model and trained to detect the objects for each world. Below are the normalized confusion matrices for each of the test worlds:
#### Test World 1 Normalized Confusion Matrix
![alt text][image4]
#### Test World 2 Normalized Confusion Matrix
![alt text][image5]
#### Test World 3 Normalized Confusion Matrix
![alt text][image6]


### Pick and Place Object Detection Results

For all three tabletop setups (`test*.world`), object recognition was performed, and then the respective pick list (`pick_list_*.yaml`) was read. The output of the detection pipeline in RViz are shown in the images below:
#### Test World 1 Object Detection Ouput
![alt text][image7]
#### Test World 2 Object Detection Ouput
![alt text][image8]
#### Test World 3 Object Detection Ouput
![alt text][image9]


Finally, the messages that would comprise a valid `PickPlace` request output them to `.yaml` format are generated in stored in the `ouput_yaml_files` directory.

### Discussion
- As we can see from the output images above, the object detection pipeline was able to detect all objects in Test World 1 and 2, but missed the glue in Test World 3. This happens because the glue is obstructed by the book, and that makes it difficult to detect. One possible solution is to have the robot complete a full rotation around the table, and hence, be able to have a clear view of obstructed objects.
- Even though the number of training instances for each dataset was high, the pipeline sometimes misdetected some of the objects. It is perhaps better to use a more complicated training model such as a Deep Neural Network to enhance detection, or simply take a majority vote over multiple detections.
- Currently, the path planning for the picking and placing of the objects is working. However, due to friction parameter and weight issues with the objects, the Gazebo environmet is not allowing objects to be gripped by the gripper. This needs to be looked into further.


  
