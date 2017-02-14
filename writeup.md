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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in calibration_utils.py.  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![Camera Calibration](https://github.com/yyporsche/CarND-Advanced-Lane-Lines/raw/master/pics/camera_calibration.png)

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

![Distortion Correction](https://github.com/yyporsche/CarND-Advanced-Lane-Lines/raw/master/pics/distort_correction.png)

####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of HLS color and gradient thresholds to generate a binary image (thresholding steps at lines 13/14, 34-39, 52-58 and 67-69 in `binarization_utils.py`).  Here's an example of my output for this step.

![Binarization](https://github.com/yyporsche/CarND-Advanced-Lane-Lines/raw/master/pics/binarization.png)

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is in `perspective_utils.py`.  The `birdeye()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```
src = np.float32([[w, h-10],    # br
                  [0, h-10],    # bl
                  [546, 460],   # tl
                  [732, 460]])  # tr
dst = np.float32([[w, h],       # br
                  [0, h],       # bl
                  [0, 0],       # tl
                  [w, 0]])      # tr
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![Perspective](https://github.com/yyporsche/CarND-Advanced-Lane-Lines/raw/master/pics/perspective.png)

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

There are two conditions for the lane-line identification: if we have a new frame, we will use sliding windows search which is implemented in line_utils.get_fits_by_sliding_windows(): we will start from the bottom, find peak locations from the histogram of the binary image, there will be two windows sliding to the upper side of the image, deciding the two lane-lines

If we confidently identified lane-lines in the previous frame, we can limit our search based on previous one which is implemented in line_utils.get_fits_by_previous_fits(). The following code is used to track the flow:

```
def __init__(self, buffer_len=10):

    # flag to mark if the line was detected the last iteration
    self.detected = False

    # polynomial coefficients fitted on the last iteration
    self.last_fit_pixel = None
    self.last_fit_meter = None

    # list of polynomial coefficients of the last N iterations
    self.recent_fits_pixel = collections.deque(maxlen=buffer_len)
    self.recent_fits_meter = collections.deque(maxlen=2 * buffer_len)

    self.radius_of_curvature = None

    # store all pixels coords (x, y) of line detected
    self.all_x = None
    self.all_y = None
```
The polynomial finding will be like:
![Lane Polynomial](https://github.com/yyporsche/CarND-Advanced-Lane-Lines/raw/master/pics/lane_polynomial.png)

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The function compute_offset_from_center() in main.py will compute the offset from the middle of the lane. We can assume the car's deviation from the lane center is same as the distance from the center to the midpoint of the two lane-lines detected at the bottom.

During lane-line detection phase, a 2nd order polynomial is fitted to each lane-line using np.polyfit(). This function returns the 3 coefficients of the curve: the 2nd and 1st order terms plus the bias. From this coefficients, following the equation from the class, we can compute the radius of curvature.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The whole processing pipeline, which starts from input frame and comprises undistortion, binarization, lane detection and de-warping back onto the original image, is implemented in function process_pipeline() in main.py.

The following showed the result for one of the test images:

![After Pipeline](https://github.com/yyporsche/CarND-Advanced-Lane-Lines/raw/master/output_images/test2.jpg)

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://youtu.be/5zjN5NDLrDQ)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I used the following [git repo](https://github.com/ndrplz/self-driving-car/tree/master/project_4_advanced_lane_finding) as starting point of the project. For image threshold, I changed from HSV in original repo to HLS to have more robust lane finding.

