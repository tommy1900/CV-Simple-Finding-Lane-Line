
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


