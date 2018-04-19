## Project: Perception Pick & Place


[//]: # (Image References)

[image1]: ./writeup_images/no_filter.png
[image2]: ./writeup_images/filtered.png
[image3]: ./writeup_images/cluster.png
[image4]: ./writeup_images/conf_mat_1.png

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
    <td class="tg-us36">Bin Range</td>
    <td class="tg-us36">[0,256]<br></td>
  </tr>
  <tr>
    <td class="tg-us36">Using_HSV</td>
    <td class="tg-us36">True<br></td>
  </tr>
</table>

To make the model robust, the number of training instances had to be large enough to provide acceptable detection accuracy. The training instances are then fed to the SVM model and trained to detect the objects for each world. Below are the normalized confusion matrices for each of the test worlds:

![alt text][image4]








### Pick and Place Setup

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.

And here's another image! 
![demo-2](https://user-images.githubusercontent.com/20687560/28748286-9f65680e-7468-11e7-83dc-f1a32380b89c.png)

Spend some time at the end to discuss your code, what techniques you used, what worked and why, where the implementation might fail and how you might improve it if you were going to pursue this project further.  


  
