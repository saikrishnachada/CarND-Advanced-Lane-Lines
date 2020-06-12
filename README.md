# **Project 2 : Advance Lane Finding** 

### Objectives 

To write a software pipeline to identify the lane lines in a video from a front-facing camera on a car and output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.  

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

---
[//]: # (Image References)

[image1]: ./output_images/chess_board_corners.png "Chess Board corners"
[image2]: ./output_images/undistorted_image.png "Distorted vs Undistorted chess board image"
[image3]: ./output_images/undistorted_image_2.png "Distorted vs Undistorted"
[image4]: ./output_images/color_thresholding.png "Color and Gradient Thresholds"
[image5]: ./output_images/perspective_transform.png "Perspective Transform"
[image6]: ./output_images/polynomial_fiit.png "polynomial fiit"
[image7]: ./output_images/radius_curvature.png "radius of curvature"
[image8]: ./output_images/projected_image.png "projected image"
[video1]: ./test_videos_output/project_video.mp4 "Video"

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

This is my writeup for the advanced lane finding project.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The overall implementation can be found in "./P2.ipynb", a python notebook file. The first code cell consists of the  useful packages for running the project. The camera calibration and distortion correction is done using chess board images. The computation involved in finding and drawing the chess board corners can be found in code cell 2. 'objp' is used to store the grid points in the (x,y,z) coordinate format. Assumption here is that the chessboard is fixed on the (x, y) plane at z=0. The 'objpoints' appends the 'objp' points every time if the corners are found in the chess board image. The detected corners are appended to 'imgpoints'. In here, all the chess board images in the "./camera_cal/calibration*.jpg" folder are used to gather the object and image points. I have used the cv2 library, specifically 'cv2.findChessboardCorners()' to find the corners and 'cv2.drawChessboardCorners()' to draw them onto the chess board images. Few examples are shown in the below image. 

![alt text][image1]

In the code cell 3, the camera is calibrated using the 'objpoints' and 'imgpoints' with the 'cv2.calibrateCamera()' function. The original image could have some camera distortion and this can ideally be corrected using the camera matrix 'mtx', distortion matrix 'dist' with the 'cv2.undistort()' function. An example is presented below in which an distorted image is corrected using the tehcniques above discussed. 

 ![alt text][image2]


### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

In addition to the above mentioned example, I have tried out the distortion correction on a test-image form the folder "./test_images/test5.jpg". I have written a function called 'undistort_image()', given in code cell 5, which takes in original image and the camera matric, distortion matrix calculated in the calibration step and correction the distortion. An example is presented as follows.

 ![alt text][image3]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I wanted to utilize the benefits of both color and gradient thresholds. This part of the code is written in code cell 6. I have written a function called 'color_thresholding()' which takes in a color image and,
(i) converts into HLS color space and a binary color threshold is applied using only the S-channel, as it picks up the lines well. The threshold values 'minimum = 150' and 'maximum = 255' are found to be satisfactory after multiple iterations. 
(ii) Sobel-x operator is applied to the grayscale image and its 'x' derivative is found to accentuate lines away from horizontal and normalized as 'scaled_sobel'. Later a gradient-x threshold is applied using the threshold 'minimum = 20' and 'maximum = 160'. Finally, both the color and gradient thresholds are combined together and the resulting edges are saved. 

The s-channel, color threshold, x-gradient threshold and the combined thresholds are compared in the below image. 

![alt text][image4]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is included in code cell 8. I have written a function called 'perspective_transform()', which takes in the masked image (code given in code cell 7), a source (src) and destination (dst) vertices. I have chosen the following as my source and destination points. 


```python
height = image.shape[0]
width = image.shape[1]
midpoint = image.shape[1]/2
left_bottom = [0,height]
right_bottom = [width-0,height]
left_apex = [midpoint-80,(height+190)/2]
right_apex = [midpoint+80,(height+190)/2]
src = np.float32([[200,height],left_apex,right_apex,[1150,height]])
dst = np.float32([[200,height],[200,0],[1150,0],[1150,height]])
```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200, 720      | 200, 720        | 
| 560, 455      | 200, 0      |
| 720, 455     | 1150, 0      |
| 1150, 720      | 1150, 720        |

I have tested the perspective transformation on a slight curvy and shadow image and the results are found to be satisfactory. The comparison is given below. 

![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In code cell 9, I have defined two functions 'find_lane_pixels()' and 'fit_polynomial()' for identification of the lane line pixels and for fitting right and left polynomials respectively. Using the histogram peaks in the warped image, the left and right lane pixel positions are classified. Using the sliding window approach windows are chosen moving upward in the image. Iterating through the windows the pixels within these windows, the good left lane pixels 'good_left_pixels' and good right lane pixels 'good_right_pixels' are identified. 

In the next step, once the (x,y) pixel coordinates of the left and right lane are identified, using 'fit_polynomial()' function each polynomial of the order 2 is fitted onto the right and left lane pixels. The fitted polynomial lines are shown in the below image.

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In code cell 11, I have defined two functions 'radius_of_curvature()' and 'draw_data()'. The former function takes in the meters per pixel in x and y dimension, the left and right lane points and calculates the left and right lane curve radius. Additionally, the position of the vehicle with respect to center lane is determined here. The latter function used this information to embed the radius and vehicle position onto the image frame. An example can be seen in the following image. 

![alt text][image7]

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the code cell 10, basically written a function 'drawLine()' to warp the detected lane boundaries back onto the original image. An example of the implementation can be seen below. 

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The problems I faced are likely due to the changing in the lighting conditions, shadows. It can be seen in the output video, that at certain point where there is obstruction due to full of shadow there is a little wobbling in the lane line detection. Other thresholding techniques must be investigated to tackle the change in lighting situations more robustly. I have implemented the sliding window approach and continued reusing it at every frame. However, skip the sliding windows step once you've found the lines could be investigated to perform the lane line detection more robustly. 
