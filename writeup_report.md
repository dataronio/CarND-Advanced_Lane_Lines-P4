# **Advanced Lane Finding** #

---

## **Advanced Lane Finding Project** ##

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

[image1]: ./output_images/distorted.jpg "Distorted"
[image2]: ./output_images/undistorted.jpg "Undistorted"
[image3]: ./output_images/straight_road.png "StraightRoad"
[image4]: ./output_images/looking_down.png "LookingDown"
[image5]: ./output_images/yellow.jpg "Yellow"
[image6]: ./output_images/white.jpg "White"
[image7]: ./output_images/saturation.jpg "Saturation"
[image8]: ./output_images/combined.jpg "Combined"
[image9]: ./output_images/histogram.jpg "Histogram"
[image10]: ./output_images/fast_histogram.png "Fast-Histogram"
[image11]: ./output_images/test2.jpg "Test2"
[image12]: ./output_images/test2-BirdsEye.jpg "BirdsEye View"
[image13]: ./output_images/test2-pipelined.jpg "Pipelined"
[video1]: ./project_video_output.mp4 "Video"


---


### Files Submitted ###

My project includes the following files and directories:

* **ALL_project.ipynb**  Jupyter notebook containing pipeline and all supporting functions.
* **/output_images**  Directory containing images processed and exported from notebook.
* **/camera_cal**  Directory containing images used in camera calibration.
* **/test_images**  Directory containing images for testing purposes.
* **project_video_output.mp4**  The project video processed by the lane finding pipeline.
* **writeup_report.md**  This report summarizing the results.

## Steps for Advanced Lane Finding Pipeline ##

### Camera Calibration ###

Image distortion occurs since cameras represent 3D objects as a 2D image.  Distortion inevitably occurs in the optical system.  We can correct for this distortion through a system of camera calibration.  We take pictures of known regular objects (chessboards) in various postions throughout the imaging field of the camera.

The code for this step is contained in the third and fourth code blocks in the "ALL_Project.ipynb" Jupyter notebook.  This code used 20 test calibration images of 9x6 chessboard images.  Each image contains a regular 9x6 chessboard in a different position throughout the image space.

We start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here we assume the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration chessboard image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time all chessboard corners are detected in a test image.  The image points, `imgpoints`, are the 2d points of the detected chessboards in the image plane. `imgpoints` will be appended with the (x, y) pixel position of each of the inside chessboard corners in the image plane with each successful chessboard detection.

We then use the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the `calibration4` test image using the `cv2.undistort()` function and obtained the following results.

Original Image:
![alt text][image1]

Distortion Corrected Image:
![alt text][image2]

### Pipeline (single images) ###

### Perspective Transform ###

The code block in notebook under "Perspective Transform" contains the code for forward and inverse transforms for the "bird's eye" or overhead perspective. A straight road image was used to tune the source data points.  The source points  are chosen to represent a rectangle when viewed from above.  From the forward perspective, the points do not appear as a proper rectangle. However, we know since the road is straight that in 3D reality they make a proper rectangular shape.  The destination points are chosen to be a middle rectangular strip in a 1280x720 image size.

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 595, 450      | 350, 1        | 
| 205, 720      | 350, 720      |
| 1100, 720     | 900, 720      |
| 685, 450      | 900, 1        |

The variables `PersT` and `InvPersT` contain the perspective transform and inverse perspective transform respectively. Warping of images in done with the function `cv2.warpPerspective()` using the appropriate transform.

I verified that my perspective transform was working as expected by inspecting the drawn points on the straight road image and then observing the same image with the perspective transform applied looking straight down at the road.   We can see this in the following images:

![alt text][image3]
![alt text][image4]

### Color Transforms and Binary Thresholding ###

I used three basic functions to create a binary thresholded image.  The function `get_saturation()` transforms an image to the HLS color space and finds saturation between the thresholds as long as lightness is greater than 100.  This is to make sure that very dark black saturated shadows are not falsely detected.  The function `get_yellow()` transforms the image to HLS color space and carefully finds the color yellow in the image.  This is used to pick up yellow lines on roads.  The same idea is used in the function `get_white()` which is tuned to pick up white colored lines.

We can see the functions in action on the perspective transformed straight road image.  Below we see the results of the `get_yellow()`, `get_white()`, and `get_saturation()` functions (left, right, bottom):

![alt text][image5]
![alt text][image6]
![alt text][image7]


The function `combined_filter()` does a logical OR on these three functions to find appropriate white or yellow or highly saturated color on the roads.

The following shows the combined filter in action:

![alt text][image8]


### Histogram Search ###

I implemented two version of histogram search.  The original `histogram_search()` function looks for image peaks in the binary image across the width of the image and then moves up the image.  Second order polynomial using `np.polyfit()` is used to parametrize the lane line.  An example of this search and line fitting can be seen in the image below:

![alt text][image9]

I also have a `fast_histogram()` search function which only looks locally around already detected lane lines.  We can see this below:

![alt text][image10]

### Radius of Curvature and Offset ###

The function `rad_curvature()` implements the computation of the radius of curvature of the lane and also calculates offset from the detected lane center.

### Pipeline Single Image Output ###

I will now describe the full pipeline for advanced lane finding.  The pipeline is implemented in the function `pipeline()` and consists of the following stages.

The pipeline works using global variables for the fit coefficients of the right and left lane.  The stored past windows of right and left lane locations and the average distance between the lanes are also kept as global variables.  An incoming image is first undistorted using `cv2.undistort()`. The perspective is then changed using `cv2.warpPerspective()` to a bird's eye perspective and the `combined_filter()` is run to enhance the lane lines.  The `histogram search()` is run at first followed by `fast_histogram()` in succeeding frames as long as there have been detected line pixels.  An exponential moving average of the distance between lines is calculated as a sanity check for bad frame detection.  If the distance between lines jumps from the average a full histogram search is run again.  Also five previous right and left lane lines are stored and the results are averaged to create the output right and left lane values.  The estimated path is drawn on the original image using the function `draw_path()`.  The computed radius of curvature and offset are computed and drawn on the frame.  The following shows the output of the pipeline on a single input of the test2 image.

First the input image of test2.jpg followed by the thresholded bird's eye view:

![alt text][image11]

![alt text][image12]

And finally we have test2 with the drawn lane path in green:
![alt text][image13]



---

### Pipeline (video) ##

The final video output of the project video appears to work reasonably well with a minimum of jitter.

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion ###

From analyzing the output of the challenge videos, I have identified a number of weaknesses with this approach to lane finding.  It is not clear that there would be an easy way to hand-craft thresholding filters that would work perfectly in all conditions.  Clearly, the results of the harder challenge video exposes the weakness in the approach from dramatic shifts in lighting and sharp curves.

A bigger issue is that in many situations there simply do not exist actual lines on the road.  Autonomous vehicles will have to navigate ambiguous areas that humans can with experience muddle through. Humans can naturally act in an appropriate manner in such difficult situations.  Possibly a better idea, instead of lane finding, is an integrated approach to scene understanding and semantic segmentation of the overall road surface.  Deep learning would clearly be the only usable technology for this approach.

A deep learning driving system would have to learn to react to other cars on the road.  Simply knowing where a well marked lane is not sufficient for safe driving.  Creating a system such as this would take an enormous amount of manpower and effort to cover all of the gray area and corner cases that experienced human drivers can handle with ease.
