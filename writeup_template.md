# **Advanced Lane Finding Project**
#### **Simon Mijares**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./md_files/camcal.png "Undistorted"
[image2]: ./md_files/singlepipecorr.png "Road Transformed"
[image3]: ./md_files/binary.png "Binary Example"
[image4]: ./md_files/warping.png "Warp Example"
[image5]: ./md_files/curve.png "Fit Visual"
[image6]: ./md_files/pipeline.png "Output"
[image7]: ./md_files/alltest.png "All image test"
[video1]: ./project_simon.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the sixth to the eleventh code block cell of the IPython notebook located in "./P4.ipynb".  

I start by testing the image visualization, then loading the first image calibration, since this image crop the last row of points so it needs different (9,5) parameters to call the calibration functions. Then all the set of calibration images with the corrects parameters (9,6). On each case preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

#### 1. Provide an example of a distortion-corrected image.
After the camera parameters were calculated with cv2.calibrateCamera and saved in a pickle file for future restore, these parameters are loaded and used to correct the distortion of a test image and then contrast equalized by the CLAHE algorithm as showed bellow (Code block cell 14):

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or others

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at code block cells 16 through 18 in `P4.ipynb`).
To begin, a combination of red and green channel were obtained based on the Rec. 709 in order to obtain an artificial yellow channel, then the sobel operator was used on this channel.
On the other hand, the original image is transformed to HLS color space to extract the Saturation channel.
After obtaining the Sat channel and the sobel yellow channel, both images are thresholded:
```
# For Yellow Sobel
thresh_min = 20
thresh_max = 255

# For Saturation channel
s_thresh_min = 155
s_thresh_max = 255
```
Then the sobel is processed with a close operation "cv2.MORPH_DILATE" to enhance its features and contribution to the final result.

This images are combined in an 'or' operation and then processed with an "cv2.MORPH_CLOSE" to fill the posible spaces within the lane lines.

Here's an example of my output for this step:

![alt text][image3]
The thresholds had to be soften to be able to work at conditions with shadows, allowing some noise in the binary image.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in the 19th code cell of the IPython notebook.  The `warp()` function takes as inputs an image (`img`), as well as source (`src`) points. I chose the hardcode the destination points in the following manner:
```python
dst = np.float32(
        [[350,0],
         [853,0],
         [853,720],
         [350,720]])
```
On the other hand, the source points were obtaining by trial and error using the `straight_lines1.jpg` file and the following code:
```python
tdrift = -18                  # Drift in the top of the trapezoid
bdrift = 28                   # Drift in the body of the trapezoid
ydrif = 20                    # Drift in y direction of the top of the trapezoid
xctr = -12                    # Center of the trapezoid
top = 654.442 - 622.491+67-2  # Start point of the trapezoid
base = 1040.87 - 265.789 - 20 # Start point of the trapezoid
src = np.float32([[622.491+xctr+tdrift, 431.432+ydrif],
         [622.491+top+xctr+tdrift, 431.432+ydrif],
         [265.789+base+xctr+bdrift, 676.387],
         [265.789+xctr+bdrift, 676.387]])
```
This resulted in the following source and destination points:

| Source           | Destination   |
|:-------------:   |:-------------:|
| 592.49  , 451.49 | 350, 0        |
| 689.44  , 451.43 | 853, 0        |
| 1036.86 , 676.38 | 853, 720      |
| 281.789 , 676.38 | 350, 720      |

I test my perspective transform and corrected by moving the parameters until it was working as expected drawing the `src` and `dst` points onto a test image and its warped counterpart verifying that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

After warp the perspective, the histogram of the lower half is obtained in order to acquire the peaks and start searching the dots that belongs to the lane line. As can be seen on the cell code block 20.

Then the images is divider in nwindows and start the search on windows of 2xmargin width and height.

Finally the np.polyfit function obtains the second order approximation of the selectect points.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In the same cell block 20 the points are scaled based on the approximation values to transform from pixels to meters based on the lane lines separation.
```python
# Define conversions in x and y from pixels space to meters
ym_per_pix = 28/720 # meters per pixel in y dimension
xm_per_pix = 3.7/503 # meters per pixel in x dimension
```
Then the np.polyfit is used again but now on meters.

After obtaining the curve coefficients in meters is obtained based on the following equation:

```math
Rcurve =[1+(dydx)2](3/2) / |d2x/dy2∣
```
#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the cell block 23. Then organized in separates functions in blocks 24 to 29. And finally mixed in a single function detect_lane(image) in the block 30.  Here is an example of my result on a test image:

![alt text][image6]

And this is the result in all the test images:

![alt text][image7]
---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_simon.mp4)
[![Training Data](https://img.youtube.com/vi/oLjYe-kLWKc/1.jpg)](https://youtu.be/oLjYe-kLWKc)
---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

One sensible features are the thresholds, since there is no indication of how to set them, and just by trial and error is difficult to have a value for every situation.
The implemented pipelane, might fail if a line pass around the lane lines.
To make it more robust, a more strict binarization function should be implemented.
