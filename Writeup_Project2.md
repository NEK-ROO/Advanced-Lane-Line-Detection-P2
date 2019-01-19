
# P2 - Advanced Lane Finding Project
---

## Overview

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

[undistortion]: ./write_up_images/undst.png 
[threshold]: ./write_up_images/threshold.png 
[threshold_fused]: ./write_up_images/threshold_fused.png
[warp]: ./write_up_images/warp.png
[warp_red_line]: ./write_up_images/warp_red_lines.png
[slide_window]: ./write_up_images/slide_window.png
[poly]: ./write_up_images/poly.png
[plot_back]: ./write_up_images/vehicle_offset.png
[realworld_scalar]: ./write_up_images/realworld_scalar.png
[vehicle_offset]: ./write_up_images/vehicle_offset.png
[video]: ./test_videos/test_videos_output.mp4

---

## Camera Calibration

### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the third code cell of the IPython notebook located in "./Advanced Lane Lines Detection.ipynb". I have mark most of the cells to specified what they are actually doing. They are seperated into two parts respectively for the first part is the aims of the code and the second part shows the results I got.

About **object points**, `nx` and `ny` are 9 and 6 respectively, so I prepared object points using `np.zeros` to initialize the array and `np.mgrid` to specify the axis of points.

**Image points** are detected by `cv2.findChessboardCorners`. And I am clear that when I got the objpoints and imgpoints, I shold map them up one by one.

Finally, `cv2.calibrateCamera` uses the mapped point pairs to calculate matrix and distortion coefficients for undistorting in the next step.



## Pipeline (single images)

### 1. Provide an example of a distortion-corrected image.

From `cv2.calibrateCamera` I got `mtx` and `dist` coefficients, so undistortion is getting convinient by just using `cv2.undistort`

![alt text][undistortion]



### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for my thresholding includes a function called `color_thresh`, `grad_thresh`, and `fused_thresh`.

For thresholding, I combined **color** and **gradient** methods.

In **color method**, I first converted the image into **HLS** channels and pick only **L channels** and **S channels**. And I set thresholds for both channels respectively and make pixels **1** in my binary images only if the values are over the both thresholds. 

In **gradient method**, images are converted into gray scale and `sobelX`, `direction`, and `magnitude` gradients are applied to detect the boundaries. I didn't use `sobelY` because when `sobelX` and `sobelY` were used together, boundaries became hardly detectable, so I prefered to save the infomation even though there are lots of noise.

Finally, I combined the two thresholded binary images using `np.logical_or`.

![alt text][threshold]
![alt text][threshold_fused]

### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `Warp`.

The `src` variable specified the area in the origin image to be warped. `dest` was the points mapped to `src` and specified the perspective after being warped (I set a `offset` to make the warped perspective contain more infomation, like the bird is flying higher).

After mapping the key points between the origin image and warped perspective, `cv2.getPerspectiveTransform` would return an `M` matrix for warping.

Finally, I used `cv2.warpPerspective` to warp the image.

*p.s. `Unwarp` was also implemented for recover the warped perspective back to the original perspective.*

![alt text][warp]
![alt text][warp_red_line]
*Two different warped images. It seems there is something wrong with the thresholding, but the warping process works well*

### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for my lane-line detection includes a function called `slide_windows` and `polynomial_find_lanes`.

`slide_windows` was kind of like an initialization function for `polynomial_find_lanes`. It is implemented to find the points within the windows centered by the lane lines. And the **poly lines** should fit the points. 
![alt text][slide_window]

`polynomial_find_lanes` fit a second-order polynomial to the points detected in current frame (except points in the first frame was found by `slide_windows`). Only searched the area beside the **poly lines** found in the previous frame with a margin of 80, and the new points would determine two new lines for me.
![alt text][poly]

### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for my curvature measuring includes a function called `measure_curvature`.

Before measuring the radius, I got the **real world scalar** to pixels. I tuned the `xm_per_pix` and `ym_per_pix` from the *straight_line1.jpg*. I calculated the `ym_per_pix` by dividing the dashed line in real world (3 meters) by the length of it (84 y-pixels). By the same way, I calculated the `xm_per_pix` by dividing the width of lane lines in real world (3.7 meters) by the length of it (631 x-pixels).

It measured the real world radius of the farest point in the image (`y_eval` == 719; Top of the image). I got a result of `test4.jpg` with **left: 628, right: 509**, which are less than **1000**.

![alt text][realworld_scalar]
*p.s. Histogram is used to locate the width of lane lines*

About **the position of the vehicle with respect to center**, I was using the fitted poly functions to get two points on both lines with same y-axis, and to do so , I could get the middle point in the real world. And to compare it to `width / 2`, finally I should be able to measure the offset. Actually, I tested it on `test1.jpg` and I got a result of *0.23 meter left of center*, which seems like a fair measurement.

![alt text][vehicle_offset]

### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code for my plotting back includes a function called `Unwarp`, which is in the front of my code (same cell with `Warp`).

All I need to do was just change the arguement positions in the `Warp`. I set `src` in `Warp` now the `dest` in `Unwarp`, and `dest` in `Warp` to `src` in `Unwarp`. Then `cv2.getPerspectiveTransform` should return an `M` that could plot the bird-eyes view back to the original one. But I thought there should be an easier way by just inversing the `M` I got from `Warp`, because $M * M^{-1}$ should be 1, and which means it is not changing the perspective.

![alt text][plot_back]

---

## Pipeline (video)

### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result][video]

---

## Discussion

### Problems and My Solution
1. I used 4min 27s to process the 'project.mp4'. Is that normal? Or it is slow?
2. `Color thresh` sometimes ruins the good detection of the `gradient thresh`, so I want to not combine the `color thresh` and only use `gradient thresh` when non-zero points in `color thresh` are too many. But as environments varies, I think it is better to implement the `thresh` **after** `warping`. 

### Further Work
* Smoothing
* Sanity Check (especially important for *challenge* and *harder_challenge*)  


```python

```
