## Part 01
 # Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

1. Import the relevant packages
2. Read in the image and select the region of interest to reduce the impact of noise
3. Read in the grayscale of the image with Gaussian smoothing
3. Tune the grayscale low/high gradient thresholds and get the Canny masked edges
4. Tune the Hough transformation design parameters and get the hough lines
5. Draw the left/right lines, average out the segment line.
6. Create a "color" binary image to combine with line image and output it as the result
![image.png](attachment:image.png)

To draw a single line on the left and right lanes, I modified the draw_lines() function the following way:
1. Separate the left-right line segments by their slope sign
2. Collect the slope and intercept of every single segment and average them out
3. Find the line y.max and y.min based on the region of interest edges
4. Regenerate the averaged line according to the obtained function in step 2 and 3
5. Draw the lines use cv2.line()

# Identify potential shortcomings with your current pipeline

The line detection algorithm is not consistent/robust enough, and the line slope oscillates when:
1. The traffic line become curved
2. There is a random white colored car overlay with the traffic line
3. Random white dots/noise in the image
I have to adjust color selection and region of interest value for different video because the line position and vehicle size/dynamic are different. However, in reality, we can't keep turning the vehicle while it is driving, the line detection algorithm should work under any circumstances. Therefore we need an On-line self-tuning/adaptive algorithm based on the image input characteristics to make the line detection more robust.

# Suggest possible improvements to your pipeline

1. The slope imperfections can be fixed by trim off the curved part of the line, and the white color car noise/disturbance can be reduced by more fine tuning of the design parameters.
2. Still, I don't think using an averaged slope for all line segments will be robust enough. I need to learn more self-adaptive/self-tuning algorithm to make the line detection more accurate.
3. The line detection equation I used is just a 1 degree/ 1st order poly equation, which means it works only for straight line detection. I can try 2nd order poly fit to make curved line detection more accurate.

# Changes made in challenge video

In the challenge video, the highway concrete wall created a "solid line" as well, which is a huge disturbance for the left line detection. Plus, there are multiple white cars continuously passing by the highway in the opposite direction, which created a disturbance for the right-line detection. I have to recreate the region array to trim off those disturbances, A triangle region of interest sidelines are too steep, I need a region sideline with less steepness, then I decided to use a polygon region with the help of function "cv2.fillPoly()". But it is still a static area, and I believe a dynamic changing region will work better.

In the video, during 4sec to 6sec, the road color changed from dark gray to white, which introduced the disturbance into the line detection algorithm. If we think about the logistic of PID controller, we need to know the historical feedback to be able to reject such kind of disturbance(sudden changes). Here I introduced "slope_previous" and "offset_previous" to smooth out the slope oscillation and a slope value saturation to filter out the abnormal slope changes.

The new draw line function - robust_line() has the following steps:
1. Separate the left, right line segments by their slope sign
2. Collect the slope and intercept of every single segment and average them out
3. Then average the current slope with the slope in the previous image frame (1-step smoother)
4. If the current slope change overshoot is higher than 30%, use only 10% of the current slope and 90% of previous slope (disturbance rejection)
5. Draw the lines use cv2.line()


## Part 02
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

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.
OpenCV functions or other methods were used to calculate the correct camera matrix and distortion coefficients using the calibration chessboard images provided in the repository (9x6 chessboard images). One of the calibration image was used to demonstrate the correctness of the calibration.
The code for this step is contained in line [3] of the IPython notebook located in "project02.ipynb". I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:
![Chessboard_cal][/CarND-Advanced-Lane-Lines/output_images/outputcalibration8.png]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![Original test image][/CarND-Advanced-Lane-Lines/test_images/test3.jpg]
The undistortion step is contained in line [5] of the IPython notebook located in "project02.ipynb". Since the on point and imgpoint is already obtained from the last section so that we can get the `mtx` and `dist` parameters from the function `cv2.calibrateCamera`. The function `cv2.undistort()` can then be used to get the undistorted image(image showing in next section).

#### 2. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.
The prespective transformation step is contained in line [7] of the IPython notebook located in "project02.ipynb". The transformation matrix M and invM are retrieved using function `cv2.getPerspectiveTransform(dst,src)`, where destination points and source points are manually picked from the image picture:
`src = np.float32([(0.42*w,0.67*h),
                  (0.61*w,0.67*h),
                  (0.17*w,0.95*h),
                  (0.88*w,0.95*h)])#low high low high
dst_offset = 250;
dst = np.float32([(dst_offset,0),
                  (w-dst_offset,0),
                  (dst_offset,h),
                  (w-dst_offset,h)])`
This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 537, 482      | 250, 0        |
| 780, 482      | 1030, 0       |
| 217, 684      | 250, 720      |
| 1126, 684     | 1030, 720     |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to confirm that the lines appear parallel in the warped image. The region of interest selection is showing in the picture below.
![undistorted and transformed test image][/CarND-Advanced-Lane-Lines/output_images/outputtest3_undist.png]

#### 3. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color(HLS and HSV), absolute Sobel threshold, directional threshold, and magnitude thresholds to generate a binary image (detailed thresholding steps at lines [8] through [10] in `project02.ipynb`).  Here's an example of my output for this step. When combining the thresholds results, I used `|` for binary color image and `&` for the rest, since the gradient methods have more noise involved.

![Binary image][/CarND-Advanced-Lane-Lines/output_images/outputBinary_test03.png]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?
The line polynomial identification step is contained in line [11] of the IPython notebook located in "project02.ipynb". A histogram peak of the image was used to find the left/right line position, and then a sliding window method was used to detect the valid/useful lane pixels in the binary image(This step is running in a loop to stack the detected lane pixels). Once a window sliding/scan is finished, the left/right line pixel positions can then feed into the `np.polyfit()` function to get the fitted polynomials. The `cv2.fillPoly()` function was used to create the polygon that illustrates the search window area.
![Polyfit image][/CarND-Advanced-Lane-Lines/output_images/outputtest3.jpg]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The line polynomial identification step is contained in line [11] of the IPython notebook located in "project02.ipynb". I defined the conversions in x and y from pixels space to meters:
`
ym_per_pix = 30/720 # 30m for 720 pixels
xm_per_pix = 3.7/750 # 3.7m for 750 pixels
`
Then use the maximum y-value(corresponding to the bottom of the image) and the fitted polynomial we got from last section to calculate for the radius of curvature based on the equation:
`
R_curve =(1+(2Ay+B)^2)^(3/2)/∣2A∣
`
The vehicle position reference is defined as the middle point of the frame(Half of the frame x-shape), and the vehicle current(real-time) position is calculated by the middle point of the left-line begining pixel and the right-line begining pixel. The vehicle driving offset(center error) can then be calculated as:`Car_reference position - Car_current position`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines [21] in my code in `project02.ipynb`. Here is an example of my result on a test image:

![Polyfit image][/CarND-Advanced-Lane-Lines/output_images/outputtest3.jpg]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](/CarND-Advanced-Lane-Lines/output_images/project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
I tested my pipeline with the challenge video(the results are terrible without any fine-tuning of the parameters). The line detection/poor polyfit failures are mostly caused by:
1. Light/shadow quick changing (sudden disturbance)
2. Bad road quality (high noise)
3. At some particular frame, the line segments may not be showing on the bottom of the screen. Plus, the window sliding scan from the bottom, it may miss some valid pixels since there were no center points at the beginning of the scan.
Some fine parameter tuning can solve some of the problems above, but since the tuning part is not dynamically changing with the video footage, I suggest to use a frame-to-frame filter that corrects the polyfit based on the line detection of the previous frame. This might be helpful for disturbance rejection.
