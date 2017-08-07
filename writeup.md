## Writeup

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


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

You can check the the camera calibration code in the Advanced-Lane-Lines.ipynb notebook.

Image distortion is caused by the projection of 3d space into a 2d image.

We have 2 types of image distortion:
1) Perspective Distortion - causes objects to look distorted/unproportional depending on the their position relative to the camera or image.

2) Optical Distortion - causes by the optical design of the lenses, light rays bend on the edges and it makes the image look curvy.

We want to undo these distortions and for that we need to calibrate the camera.
Basically we need a known size object so we can compare a "real world" image against the distorted images and calculate the "formula" to undistort images.

For that we will use a chessboard since it has easy high contrast patterns that we can detect(the inner corners).

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration matrix and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the a chessboard image using the `cv2.undistort()` function and obtained the result that you can see in the notebook.

### Pipeline (test images)

Check the notebook for visualization of the pipeline on test images

#### 1. Provide an example of a distortion-corrected image.
Second Image @ Notebook

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (check `color_gradient_treshold` method in the notebook).

My main concern was to minimize the impact of vertical lines that are not part of the lane lines.

I spent hours trying to perfect it, I was able to get to amazing results on some images((HLS s & HLS h & gradient y) | white_mask(gray > 200)) but couldn't make it generalize well.

In the end I used a general technique bitwise_and with vertical and horizontal gradient thresholding to detect the vertical lane lines + bitwise_and with s channel of HLS color space and v channel of HSV color space.

s channel is a measurement of colorfullness and v is a measurement of lightness/brightness of color.

Please refer to notebook to see the transformation on test images.

Third Image @ Notebook

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Perpective transform is needed to avoid the distance distortion(lanes near horizon), we transform the image to a "bird view".

The code for my perspective transform is in the method `perspective_transform_image()` in the notebook.

The function takes a gray image, source points and destination points.

source points and destination points should be in the same order.

basically I'm converting a trapezoid around the lines to a rectangle spread all over the image.
    
    img_size = (gray.shape[1], gray.shape[0])
    
    x_offset = img_size[0] / 4 # x offset for dst points
    y_offset = get_horizon(gray) # y offset for dst points
    center_x_offset = x_offset / 6;
    bottom_y_offset = 35 # pixels - remove car hood
    
    src = np.float32([[img_size[0]/2 - center_x_offset,y_offset], \
                     [img_size[0]/2 + center_x_offset,y_offset], \
                     [x_offset,img_size[1] - bottom_y_offset], \
                     [img_size[0] - x_offset,img_size[1] - bottom_y_offset]])
     
    dst = np.float32([[x_offset, 0], [img_size[0]-x_offset, 0], \
                     [x_offset, img_size[1]], [img_size[0]-x_offset, img_size[1]]])


This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 587, 720      | 320, 0        | 
| 693, 720      | 960, 0        |
| 320, 685      | 320, 720      |
| 960, 685      | 960, 720      |

I verified that my perspective transform was working as expected by testing it on test images and see that the lines are mostly parallel.

Fifth Image @ Notebook

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I did all that in a method named find_lines_fit() in the notebook.

We now have the lanes from a "bird view" and we want to fit a polynomial to each lane.

If we have no prior high confidence measurement for the lines, we take an histogram of the bottom of the image in order to identify peaks of white pixels which are usually the center of the lane.

We take the largest peak on each side and link it to a lane, we have some offset from the middle to avoid noise in the middle since the lines are on the sides.

When we get the base of the line, we use a sliding window to go up and search for the line.

In each window we look for positive/white pixels and append them to a list, and we re center the window if the number of pixels is above a certain threshold.

I use some kind of momentum as well to shift the windows on the lines direction.

if we do have a prior measurment, we don't have to search blindly for lines again, we can just search around the old fit.

we extract x and y pixels for each lane and fit a second order polynomial to each.

Sometimes we don't have enough points, or we identified the wrong points so the second order fit is off.

Sixth Image @ Notebook

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

To understand the calculations behind the radius of curavature, please go to: http://www.intmath.com/applications-differentiation/8-radius-curvature.php

get_radius_curvature() and get_car_offset were used to calculate the radius of curvature of both lanes and the position of the vehicle with respect to center.

The lanes fit were converted from pixels to meters by fitting polynomials again in "real world" and then computing the radius of curvature from equation with the polynmials coefficients.

Position of vehicle was calculated at the base of the vehicle.
The camera is at the center of the vehicle and we have the base of the lanes from the polynomial fit.
So we can compare the middle of the lanes to the camera position and know the vehicle position relative to the center.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the method draw_driving_space().
It draws a green polygon filling area between the lines.

Seventh Image @ Notebook

You can see it drawn on the video with the radius of curvature and vehicle position.

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](test_videos/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.

The main problem was removing/thresholding all the noise(stuff other than the lanes, mostly vertical noise) from the images, couldn't make anything generalize to all situations.

The magnitude threshold didn't work as I expected but I'm sure with enough time I can remove "thin" vertical lines which are not the lane lines.

I'm sure it can be improved, we can have different processing for specific types, like we can detect if an image is dark and run histogram equalization and other techniques.

Should use more color masks like white and yellow and thresold the lightness/brightness of it as well.

My pipeline will fail under many light conditions including shades and strong light.

Another problem was the the perspective transformation, the lanes weren't parallel in all images, which hurts detection and means I can't rely on it and assume other things from it.

I do have sanity checks to filter incorrect identifications, I rely on prior high confidence data.
Since an image can pass sanity checks but still be not fully correct, I implement data smoothing so changes will be gradual.

The sanity checks include checking that the lanes are in a feasible distance from each other, should be 3.7 meters or 700 pixels, I gave it a bigger threshold since the lines aren't always parralel in the perspective transform.
Since the offset is measured from the base of the vehicle, it was a good measurement.

The radius of the curvature is absolute value was not a good measurement, as it varies a lot, sometimes it would go high up since the lanes fit is far from perfect.
The relative measurement was a good indication of the lanes.

I noticed that the sliding window after we have a fit doesn't work as good, many times the fit had to reset and search from scartch again, it would work better when the the rest of the pipeline and therefore the polynomial fits would be better.
