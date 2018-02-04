## Advanced Lane Finding Project

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

[undistorted]: ./output_images/undistorted.jpg "Undistorted"
[incomplete]: ./output_images/incomplete.png "Incomplete"
[undistorted2]: ./output_images/undistorted2.png "Undistorted2"
[pixelfeatures]: ./output_images/pixelfeatures.png "pixelfeatures"
[warpperspective]: ./output_images/warpperspective.png "warpperspective"
[polyfitting]: ./output_images/polyfitting.png "polyfitting"
[lanepolygon]: ./output_images/lanepolygon.png "lanepolygon"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Introduction

All the code is contained in P4.ipynb notebook, which requires conda `carnd_term1` environment packages to execute.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the code cell no.3 of the notebook.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Initially, these coordinates are populated in the array `ptsObj`. Every time I successfully detect all chessboard corners in a test image, I append the image coordinates to the array `ptsImg`, and grow `ptsObj` array by one element.

I then used these arrays to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![][undistorted]

The result is not perfectly undistorted, but this is the best we can get with the given amount of calibration images, especially provided some of them did not pass the chessboard corners extraction step, as an example below:

![][incomplete]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![][undistorted2]

As can be seen, the difference is barely noticeable, except the white car in the undistorted (right) image is stretched outside of the image boundary. The effect of this step can be easier obserived with fisheye kind of lens.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Cell 5 of the notebook contains all the code and magic thresholds for pixel-level features and the extraction procedures. I chose to use a combination of color and gradient orientation+magnitude thresholds to generate the binary feature map. Here is an example on the sample image:

![][pixelfeatures]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Cell 7 of the notebook contains code for the perspective transform, mapping the trapezoid region of interest with lane lines to a rectangular image of the same size as an input image. The trapezoid coordinates were chosen using the sample image below, containing straight lines, such that the warped image lane lines become parallel. The result should allow for moderate road curvature detection with no sharp bends, so the margins on each side are roughly 0.5 lane width.

![][warpperspective]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Cell 9 of the notebook contains code for:
- initial detection of the left and right lane lines using the sliding window approach, which also fits a 2nd order polynomial to the detected pixel groups (function `fitLineInitialization`)
- refinement of the lane lines positions, provided the last frame's polynomial coefficients (function `fitLineApproximation`)

When working with standalone images, only `fitLineInitialization` is used. It effectively applies the sliding window approach with histograms to detect the lane lines. Limitations of this approach are discussed in the last section of this report.

For the case of video input, the simple strategy is to use `fitLineInitialization` on the first frame, and also eventually start over every N seconds or frames, to make sure the lane detections do not drift away. Other frames should be treated with `fitLineApproximation` function, provided the polynomial coefficients of lanes in the previous frame. It uses the input polynomial coefficients to produce the zones, which are likely to contain lane pixels, then simply fits polynomial coefficients to refine.

Both functions consist of 3 parts:
- finding the set of candidate pixels
- fitting polynomial to the sets of candidate pixels, corresponding to the left and right lane lines
- visualization

![][polyfitting]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The same cell 9 also contains the code of curvature approximation. I think the lecture notes avoid discussing a lot of potential issues related to the scale of the curvature results, which inevitably depend on the warped image size, camera mounting height, and trapezoid anchors selection. The result can only be as good as +/- order of magnitude.

For the `test2.jpg` image, the curvature was calculated as follows:
```
Lane curvatures in meters: left=3875.00 right=3875.00
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Cell 11 contains the code for mapping the area between detected lanes onto the original (yet distortion-corrected) image. Instead of drawing a polygon using `cv2.fillPoly` function, using the coordinates sampled from the lane polynoms, and subsequent warping the image back using the inverse perspective transform, I rather convert the polygon vertex coordinates back into the source frame of reference, and draw a polygonal overlay directly in the source space.

![][lanepolygon]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a visualization (youtube) of all stages of the pipeline performed on the project video:
[![](https://img.youtube.com/vi/JQoi-KP0LWg/0.jpg)](https://www.youtube.com/watch?v=JQoi-KP0LWg)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Although I used the suggested approaches with using a robust hypothesis from the previous frame as an approximation for the current frame, and low-pass filtering, still, this framework is highly tuned towards a vanilla case of sunny weather and clean roads. The following issues inevitably arise:

- Color based lane feature extraction depends on the lighting conditions, which fluctuate greatly. During night time, grayscale is all that can be accounted for.
- Gradients can fail due to image noise or high frequencies of any other nature
- Lanes may have long durations of invisibility, which will be a hard case to recover from with this pipeline
- The challenge videos (attempts made at the end of the notebook) require more tuning to operate:
    - expand the rectification area to allow detection of sharp bends with high curvature
    - drop the histogram based method in favor of something else. I would try something along the lines of constrained hough transform, with the following constraints:
        - both lines have very common polynomial coefficients, the only major difference is the offset
        - both lines are known to intersect the bottom boundary of the frame at predictable positions
    - drop color as criterion to reduce feature map clutter

End of report
