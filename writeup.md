# Advanced Lane Finding Project


The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

NOTE: The code used to generate the example images is at the end of the ```code.ipynb``` notebook.

[//]: # (Image References)

[image_calibration]: ./output_images/calibration.png "Calibration"
[image_original]: ./test_images/test5.jpg "Original road image"
[image_undistorted]: ./output_images/undistorted_image.jpg "Undistorted"
[image_perspective_transform]: output_images/perspective_transform.png "Perspective Transform"
[image_perspective_transformed]: output_images/perspective_transformed.png "Perspective Transformed"
[image_thresholded]: output_images/thresholded.png "Thresholded image"
[image_warped]: output_images/warped.png "Warped image"
[image_thresholded]: output_images/thresholded.png "Thresholded image"
[image_threshold_test]: output_images/threshold_test.png "Threshold test images"
[image_polynomial_fitting]: output_images/polynomial_fitting.png "Polynomial Fitting image"
[image_final]: output_images/final.png "Final result image"


## Camera Calibration

The code for this step is in the "Camera Calibration" section, in code box 3 of the ```code.ipynb``` IPython notebook. This follows the code from the lectures:

* We first use the cv2.findChessboardCorners() to find the points of the chessboard in each image (the imgpoints).  This will give us the 2D points in the image for the location of the chessboard corners.
* These are then matched together with the corresponding points in objective space (the objpoints, which will be the same for each image). The objpoints are 3D points.  This can be arbitrary so we choose z=0 and x,y points that are at integer corners ( (1,1), (1,2), etc...)
* This is then run through the cv2.calibrateCamera() function to generate the camera matrix and distortion coefficients that will be used to distort/undistort the image.

Here is a example of the working code.  The original image is shown on the left and the undistorted image is shown on the right.

![calibration example image][image_calibration]

## Pipeline (single images)

In the following sections, we will look at the processing of a single image to explain the processing that is being done.

### 1. Original Image

This is one of the images supplied in the "test_images" folder. This is one of the images that I used for calibration due to the difficulty of making out the lane lines due to (1) the tree shadow and (2) the change in color from the concrete to the asphalt.

![original image][image_original]

### 2. Distortion Corrected Image

This is the image after it has been run through the undistort code to remove distortions caused by the camera.

![original image][image_undistorted]

### 3. Perspective Transform

I then ran a perspective transform to normalize the images.  I found this easier to do before thresholding, as it made it easier to find some of the features (the lanes further away were more difficult to find).

The code for the perspective transform is found in the "Perspective transform" section of the ```code.ipynb``` notebook.  The perspective transform relies on two rectangles, the source rectangle and the destination rectangle.

These were found by taking one of the sample images and marking the lane.  This gave us the dimensions of the source rectangle.

The destination rectangle coordinates were then chosen to be roughly the shape of the "true" rectangle.  This was determined by trying to guess how far the lines extended in the real world (using some sizing of the length of the short lane lines).

This resulted in the following source and destination points:

| Location           | Source        | Destination   | 
|:------------------:|:-------------:|:-------------:|
| upper left line    | 555, 480      | 320, 0        | 
| upper right line   | 735, 480      | 980, 0        |
| lower right line   | 1060, 680     | 980, 720      |
| lower left line    | 245, 680      | 320, 720      |

The code that initializes the points is in the "The LaneFinder object" section in ```code.ipynb```.

Note that this is a different image from the examples.  The reason is that we need an image with straight lines in order to calibrate the transform.

![Perspective Transform][image_perspective_transform]

The actual transform is performed using the cv2.warpPerspective() functions using the transforms provided from the cv2.getPerspectiveTransform() functions. The code for this is encapsulated in the ```PerspectiveTransform``` class in the "Perspective Transform" section of the ```code.ipynb``` IPython notebook.

![Perspective Transformed][image_perspective_transformed]

I have drawn two parallel red lines to show that the image has been transformed correctly.  In the code, you can see that I have used the same coordinates that were used to perform the actual transformation.


### 4. Thresholding

The code for the individual transforms can be found in the "Basic Thresholding Algorithms" section in ```code.ipynb``` IPython notebook.  The final transform (which is a combination of methods) can be found in the "Final Thresholding Transform" section of ```code.ipynb```.

Here is the original image (after undistorting and perspective transformation).

![Warped original image][image_warped]

Here is the final thresholded binary image.  After some experimentation, I used a combination of transforms:

* a filter of the S channel from the HLS color space (values must be within 150-255)
* a filter of the S channel from the HSV color space (values must be within 200-255)
* a filter using the Sobel operator in the x axis (kernel size = 5 and thresholds of 10-250)

![Thresholded image][image_thresholded]

The next image is a test image I generated to look at how the various transforms performed. The images may not match the real thresholding algorithm due to adjustments made to the parameters.  this is provided as an example of how I decided on the final method.

* The first two rows uses the S channel from the HLS color space.
* The third and fourth rows use the S channel from the HSV color space.
* The fifth row shows some the results of combining the transforms.

![Threshold test][image_threshold_test]

### 5. Lane-line Pixels and Polynomial Fitting

The Lane-line pixel detection and polynomial fitting code can be found in the "Fitting a Polynomial" section in the ```code.ipynb``` IPython notebook.

The pixel detection code for a new image is in ```PolynomialFitting.fit_to_image``` lines 18-83. We take a histogram of the points in the bottom half of the image to determine the lane midpoints.  We then use a window to determine the lane points and move it upward to find all the points in the lane.

The last bit of code in ```PolynomialFitting.fit_to_image``` lines 85-97 does the actual work of taking the points and fitting a quadratic curve to the points.

Here is a test image that shows the windows and the fitted curve.

![Polynomial fitting image][image_polynomial_fitting]

There is also additional code in ```PolynomialFitting.next_image_fit``` which uses information from the previous fit to fit the curve (it doesn't attempt to relocate the curve centers).

### 6. Curvature and Vehicle Center Calculations

The radius of curvature calculations for a lane can be found in ```LaneFinder.calculate_curvature``` in the ```The LaneFinder object``` section of the ```code.ipynb``` IPython notebook.

For the curvature calculations, I use the polynomial for the curve, evaluate it in pixel space, and then fit it to a new curve in world space (meters).  We then find the new radius at the bottom of the image.

The Vehicle Center calculation is an average of the left lane center and the right lane center (in pixel space). The difference this is from the center is then calculated and converted to world space (meters). The constants for the conversion from pixels (pixel space) to meters (world space) can be found in ```LaneFinder.__init__``` lines 18-23.

These calculations can be found in:

* PolynomialFitting.fit_to_image lines 72-79
* PolynomialFitting.next_image_fit lines 134-140



### 7. Final result (putting everything together)

The code to process a single image is in ```LaneFinder.process_image``` in ```The LaneFinder object``` section in ```code.ipynb```.

This will take an image and then draws out the lane that was detected with additional text output also (the radius and lane center).

The code in ```LaneFinder.determine_fit``` implements the control logic when a bad image is detected.

The code in ```LaneFinder.sanity_checks``` does the checking to determine if the values obtained from an image are valid and can be used.

![Final image result][image_final]


---

## Pipeline (video)

Here's a [link to my video result](./project_output.mp4)

---

## Discussion

Here are some things that came up when developing the solution.

#### 1. Correct dimensions for the warp

Determining the proper sizing of the rectangle used for the warp is very important (this is also used
Had to determine the proper constants for pixel-to-meter conversion). If this is off, it can lead to lines that expand as they get nearer to the top of the image.

This could be calibrated more exactly if the exact dimensions were known (or if we had control of the camera).

#### 2. Window sizes

Determining the proper size of the number of frames to average over was challenging. I didn't want to make it too large but had to make it large enough to cover cases where the shadows were interfering in the image.

This leads to issue 5, resetting the image data in a bad scene can lead to bad outcomes.

#### 3. Image fitting on curvy roads

On very curvy roads, such as in the second challenge video, initial image fitting can be difficult.

I think this is true because the sliding window algorithm may loose track of the lane line as they become more horizontal than vertical.

One approach may be to use overlapping windows, so that the algorithm can keep track of the lines better.

#### 4. Better thresholding

Determining the thresholding was rather time consuming as there are a lot of variations that need to work in various situations.

A wider variety of sample images would be needed to determine a better algorithm (as evidenced by the challenge videos).

I could use the approach from the first lane fitting problem to remove some of the random lines (which may help to remove the effects from tree shadows).

#### 5. Initial image fitting

The program is very sensitive to the initial image fitting.  If the code resets because of a series of bad images, the bad image may then be used as the basis for the lane lines.

There was nothing much that I could do here except to increase the number of images that we average over, or to increase the number of bad images allowed.

#### 6. Sanity check parameters

This also took quite a while to determine how bad an image needs to be before rejecting.

Doing some analysis may be useful to determine the allowable differences between the left and right lanes curvatures.

#### 7. Curvature calculations

My curvature calculations were higher than expected. After much experimentation, I found out that it was due to my perspective transform being a bit short, so my curves were straighter than expected (since I had less data with the dashed lines).  However, being able to see further ahead led to the algorithm being much more instable (especially in the bridge area and when the black car passed me).

So I added a parallel calculation for the radius of curvature.  I basically do two perspective transforms, use one for the curvature and one for the lane detection.  A possible optimization of this would be to use one transform and have the lane calculations truncate the image to the size it expects, however I did not have time to try this out.