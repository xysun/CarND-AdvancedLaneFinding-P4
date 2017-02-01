##Project 4 Writeup
---

**Advanced Lane Finding Project**

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

[image1]: ./output_images/undistort_example.png "Undistorted"
[image2]: ./output_images/input_1.jpg "Original road image"
[image3]: ./output_images/input_1_undistorted.jpg "Original road image - undistorted"
[image4]: ./output_images/input_1_color_threshold.jpg "Color threshold example"
[image5]: ./output_images/input_1_regional_mask.jpg "Regional mask example"
[image6]: ./output_images/straight_lines1.jpg "Original straight line image"
[image7]: ./output_images/straight_lines1_warped.jpg "Warped straight line image"
[image8]: ./output_images/input_1_warped.jpg "Warped road image"
[image9]: ./output_images/input_1_fit_polynomial.png "Polynomial fit"
[image10]: ./output_images/input_1_final.jpg "Final result"


[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the IPython notebook located in "P4 Pipeline.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

Since we only need to calibrate camera once, the calibration results (camera matrix `mtx` and distortion coefficients `distort`) are saved as pickle, so that in the future we only need to reload from pickle files.

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the road images from project video like this one:
![alt text][image2]

The code for undistort can be found in code cell #4 in iPython notebook "P4 Pipeline": applying camera matrix and distort coefficients obtained from camera calibration step in the function `cv2.undistort(test_img, mtx, dist, None, mtx)`.

Here's the output image, you can see it is slightly "stretched" after the distortion-correction step.
![alt text][image3]

####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
To obtain a thresholded binary image, I used only color thresholding, but with two different spaces: R from RGB and U from [LUV](https://en.wikipedia.org/wiki/CIELUV). In particular, I found LUV is more robust for shadow areas than HSV.

The code for color thresholding can be found in cell #5 in "P4 Pipeline" notebook. The threshold I used are:

- R: (220, 255)
- U: (120, 255)


Here's an exmple of output for the same road image as above for this step:

![alt text][image4]

After color threshold, I also applied a regional mask to exclude irrelevant portions of the image. I used same code from Project 1. The code is in cell #6 in "P4 Pipeline" notebook.

Here's an example of output after regional map:

![alt text][image5]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Code for my perspective transform can be found in cell #9.

I chose to manually pick the source and destination points from straight_lines1.jpg

The chosen points are as below:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 765, 504      | 800,450       | 
| 1060,700      | 800, 700      |
| 240,  700     | 200, 700      |
| 525, 500      | 200,450       |

I verified that my perspective transform was working as expected by drawing the warped counterpart to verify that the lines appear parallel in the warped image.

Here's the original image:
![alt text][image6]

Here's the warped image for the lanes from the straight line image, they're pretty well parallel:
![alt text][image7]

Here's the warped image for the road example above, two lanes are clearly visible: 
![alt text][image8]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I used same methodology as in the [lecture](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/c41a4b6b-9e57-44e6-9df9-7e4e74a1a49a), namely, take a histogram along columns of the lower half of the image; find the peaks, then apply a series of sliding windows around each peak (left and right) to find the lane pixels, and finally fit a polynomial to the pixels.

The code can be found in function `histogram_search()` in cell #13. `histogram_search()` return a tuple of found pixels: `(leftx,lefty,rightx,righty)` that are later used in `Lane.get_fit()` function to derive the final polynomial fit.

Here's a sample output demonstrating this step for the road image example:
![alt text][image9]


After fitting the first image, we can base subsequent searches from the previous pixel position, i.e instead of searching blindly from histogram peaks, we can search within margin of the previous lane position.

The code for this search can be found in function `histogram_search_next()` in cell #14.

In `Lane.get_fit()` function, a polynomial fit is only deemed as a "good fit" if it is: 

- left and right lanes are parallel, i.e the highest coefficient is of same sign
- curvature is more than 200m. This is to filter out cases where too few pixel points result in a drastic polynomial fit. 

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated lane curvature through function `get_curvature()`. This is following same approach as in [lecture](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/2f928913-21f6-4611-9055-01744acc344f) and transformed from pixel space to world meter space.

For vehicle position, code is in function `get_off_center()`. I take the bottom pixels of fitted left and right polynomial, transform them back using the perspective matrix, and compare the middle point of the transformed pixels with screen's mid point. 

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Code for plotting back is in function `draw_back()`. I followed same approach as in [lecture](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/7ee45090-7366-424b-885b-e5d38210958f), the only difference is I applied curvature and vehicle position text on top of the final image as well.

To smooth measurement, the actual lane drawn back onto the image is the average of last 3 "good fits" (see section #4 on criterion of a good fit). The smoothing code is again in `Lane.get_fit()` function. 

Here's the example of applying all the steps for that road image:

![alt text][image10]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

