
**Vehicle Detection Project**

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector. 
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[image1]: ./examples/example_car.png
[image2]: ./examples/example_color_features.jpg
[image3]: ./examples/example_heat.jpg
[image4]: ./examples/example_hog.jpg
[image5]: ./examples/example_svc.png

[video1]: ./project_video_out.mp4

## [Rubric](https://review.udacity.com/#!/rubrics/513/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Vehicle-Detection/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

The project comes with two notebooks, the first was used to generate example images for the report and to train a classifier, the second contains the pipeline for processing the video.

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.
I started with some exploratory analysis of the data, an example of car and not-car images from the default dataset is shown here:
![alt text][image1]
In order to improve the performance of the classifier I attempted to include the two additional datasets by extracting the bounding boxes and randomly selecting boxes not showing cars. Unfortunately
this degraded the performance of the classifier (probably a bug somewhere in the pipeline), so I ended up sticking to the default data.

In the first notebook HOG features are extracted from test images and later applied for training a linear SVC. The functions are the ones introduced in class. A visualization of the hog features is displayed here:
![alt text][image4]

Later in the notebook I use the suggested combination of HOG feature extraction and sub-sampling to avoid redundant feature extraction for the classification. The same function is used to process the frames of the 
project video. Since the function converts the colorspace using CV2 both training images and frames from the video end up having the same scale (0-255).

Additionally, color features and spatial features were extracted as shown in class and concatenated to the HOG features. Here is an example image showing color histograms for one of the test images:
![alt text][image2]


#### 2. Explain how you settled on your final choice of HOG parameters.

After building a working pipeline I quickly ran into problems with false positives and tried tuning parameters, including the hog parameters. Later I understood that the issues are related more to the training
data and therefore used the same HOG parameters suggested in class, which worked quite well after the training data issued had been resolved. I used all color channels, orient of 9, 8 pixels per cell and 2 cells per block.
The one HOG paramter that did seem to have a stronger impact was the choice of color space. Both HSV and YCrCb performed better than RGB, I eventually used YCrCb.

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).
The classifier uses a combination of HOG, color and spatial features, as shown in class.

Due to issues with false positives I spent quite a bit of time experimenting with various classifiers and modifications to the training data. Initial attempts to modify the training data did not work out and the non-linear 
kernels were too slow. However, scikit learn's CalibratedClassifierCV functionality was useful for getting a probability estimate from the linear SVC, which helped reducing false positives.
Hypterparameter tuning using gridsearch showed that C=0.001 yielded best results. 
Poor results of the pipeline eventually made me realize that the training data is at fault - the balanced dataset does not work well for a pipeline that has mostly non-car images. By doubling the number of non-car images
and adding manually collected slices of problematic ares (road in some color conditions and concrete barrier on the left) the classifier's performance on the video could be improved.

The final classifier has an accuracy of 0.9942, which is a bit misleading as tests on images and video still reveal a relatively large number of false positives (that could be removed with thresholding).


### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

I used the sliding window with HOG sub-sampling as suggested in class with several scale parameters, varying y-position starting points and a minimum x-position value. The upper half of the image
was ignored due to an unfortunate lack of flying Deloreans in the project video. Despite the sub-sampling the performance on the video is still quite bad, probably some parameter tuning may help 
with reducing the number of scale parameters and thus the number of windows extracted from each frame.
The following image shows the results of sliding windows (with scale parameter 1.5) after classification, including some false positives.

![alt text][image5]

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

As described above, the video pipeline classification performance was mostly improved by augmenting the training data. As there were still false positives
I used heatmaps and thresholding to keep only high-confidence windows. The following image shows an example of thresholding on a heatmap on a test image with an early version of the classifier,
which did not perform all that well:
![alt text][image3]

For the video I took only reasonably high-confidence windows (predict_probab of 0.75 or more) and stored 10 frames in a queue and applied a threshold of 6 before labelling the boxes. Additionally,
I checked the distance from each new box to the thresholded boxes from the last frame to filter spurious short-term detections somewhere along the border of the image (every couple of frames this 
step is ignored to have a fresh starting point for detection).
---

### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)
Here's a [link to my video result](./project_video_out.mp4)
The box position could be a little more consistent (e.g. smoothing) and react faster to new objects (queue perhaps too long).

#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

I only kept reasonably high confidence detection (predict_proba > 0.75) and stored them in a queue. The boxes were added to a heatmap and thresholded to produce one larger bounding box per vehicle. I also used the distance to new boxes to reject false positives as described above.



---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
The largest problem was the bad performance of the classifier, probably related to lack of training data.

While the pipeline works it is customized to this one video. It seems that the classifier is not very robust, so even small changes in conditions are likely to be a problem. Additionally, the training set is relatively small. To increase robustness more training data (highway, urban traffic, different weather conditions, crossing roads etc.) and neural networks for object detection would probably 
be a good idea.

