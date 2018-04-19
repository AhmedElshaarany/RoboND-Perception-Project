## Project: Perception Pick & Place


[//]: # (Image References)

[image1]: ./writeup_images/no_filter.png
[image2]: ./writeup_images/filtered.png

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
#### 2. Complete Exercise 2 steps: Pipeline including clustering for segmentation implemented.  

#### 2. Complete Exercise 3 Steps.  Features extracted and SVM trained.  Object recognition implemented.
Here is an example of how to include an image in your writeup.

![demo-1](https://user-images.githubusercontent.com/20687560/28748231-46b5b912-7467-11e7-8778-3095172b7b19.png)

### Pick and Place Setup

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.

And here's another image! 
![demo-2](https://user-images.githubusercontent.com/20687560/28748286-9f65680e-7468-11e7-83dc-f1a32380b89c.png)

Spend some time at the end to discuss your code, what techniques you used, what worked and why, where the implementation might fail and how you might improve it if you were going to pursue this project further.  


  
