
---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms to create a thresholded image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image0]: ./images/test01.jpg "Undistorted"
[image1]: ./images/test01.jpg "Undistorted"
[image2]: ./images/undist_1.jpg "Road Transformed"
[image3]: ./images/warped_sample.png "Warp Example"
[image4]: ./images/warped_10.jpg "Warp Example"
[image5]: ./images/white_thresh_10.jpg "white threshold"
[image6]: ./images/yellow_thresh_10.jpg "yellow threshold"
[image7]: ./images/compute_lane.png "compute lane"
[image8]: ./images/compute_lane_2.png "compute lane 2"
[image9]: ./images/draw_4.jpg "draw mask"
[video1]: ./project_video.mp4 "Video"
[video2]: https://www.youtube.com/watch?time_continue=25&v=siAMDK8C_x8 "Slide window"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Camera Calibration

#### 1 . Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the code cell `Camera calibration` of the IPython notebook located in "./Lane_Finding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. Calibration result is saved in a pickle file "./cal_cam.p", such that I only have to do camera calibration once for all.

I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

Here are images before and after undistortion
![alt text][image0]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The code for this step is contained in the code cell under `Distortion correction` of the IPython notebook

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image1]

First load camera calibration result from pickle file "./cal_cam.p", then use `cv2.undistort()` to undistort an image.
Here is the result:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for this step is contained in the code cell under `Perspective transform` of the IPython notebook

The `warper()` function takes as inputs an image (`img`), and a second parameter `reverse`, which transform from bird view to original perspective when it is set to `True`.  I chose the hardcode the source and destination points in the following manner:

```
# set up box boundary
xmid = xsize/2 # middle point
upper_margin = 85 # upper width
lower_margin = 490 # lower width
upper_bound = 460 # upper value of y
lower_bound = 670 # bottom value of y
dst_margin = 450 # bird view width

# source points
p1_src = [xmid - lower_margin, lower_bound]
p2_src = [xmid - upper_margin, upper_bound]
p3_src = [xmid + upper_margin, upper_bound]
p4_src = [xmid + lower_margin,lower_bound]
src = np.array([p1_src, p2_src, p3_src, p4_src], dtype=np.float32)

# distination points
p1_dst = [xmid - dst_margin, ysize]
p2_dst = [xmid - dst_margin, 0]
p3_dst = [xmid + dst_margin, 0]
p4_dst = [xmid + dst_margin, ysize]
dst = np.array([p1_dst, p2_dst, p3_dst, p4_dst], dtype=np.float32)

```
This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 555, 460      | 150, 0        |
| 150, 670      | 150, 720      |
| 1130, 670     | 1130, 720     |
| 725, 460      | 1130, 0       |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this step is contained in the code cell under `Color threshold` of the IPython notebook

I used only color thresholds to generate a binary image. I generate binary image for white color lane line and yellow lane line separately. The process start by transforming the image to HSV color space using `cv2.cvtColor()`, then applying threshold to 3 channels simultaneously using `cv2.inRange()` to generate binary image.

The major challenge here will be to make the process robust to different light condition. To overcome this challenge, I use different thresholds for different light condition.

Note that a correct detection of number of lane line pixel of a color will be within a reasonable range (not too low nor too high). This fact will be used to design robust color lane detection.

Main characteristic of white color in HSV is low saturation, a good range in normal light condition is `[0, 0, 200]` to `[255, 25, 255]`. However, in low light condition this range will filter white color with low value, thus we add a second range from `[0, 0, 170]` to `[255, 20, 190]`. This low light range will give tremendous false detection in normal light condition. Thus when normal light range detect enough pixels or low light range detect too many pixels, the result of low light range will be set to zeros. In this way we can make the white color detection robust to different light condition.

Main characteristic of yellow color in HSV is that hue value is around 15 and high saturation. Then the same process of white color detection applies to yellow lane line detection, with 3 different thresholds:

`[ 0, 100, 100]` to `[ 50, 255, 255]` (normal light)

`[ 10, 35, 100]` to `[ 40, 80, 180]` (low light)

`[ 15, 30, 150]` to `[ 45, 80, 255]` (high light)


Here are examples of my output for this step.  
(different color indicates result of different threshold)

1.warped image:
![alt text][image4]
2.white lane line
![alt text][image5]
3.yellow lane line
![alt text][image6]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for this step is contained in the code cell under `Locate lane line` of the IPython notebook.

All method regarding lane tracking resides in a class name `Lane()`, as follows:

```
# Define a class to receive the characteristics of each line detection
class Line():
    def __init__(self):
        # was the line detected in the last iteration?
        self.detected = False
        # the last n fits of the line
        self.recent_left_fit = []
        self.recent_right_fit = []
        #polynomial coefficients for the most recent fit
        self.current_left_fit = [np.array([False])]
        self.current_right_fit = [np.array([False])]

    # return best fitting coefficients, which average the last 3 True dection
    def get_fit(self):

    # calculate and return deviation from lane center
    def get_deviation(self):

    # calculate and return lane curvature
    def get_curvature(self):

    # input 3 channel warped image: warped[yellow,white,sobel]
    # calculate fitting coefficients
    # set to self.current_left_fit and self.current_right_fit
    def find_lane(self, warped):

    # input 3 channel warped image warped[y,w,s]
    # use slide window method to search lane lines
    def blind_search(self, warped):

    # input 3 channel warped image
    # use best fit coefficients to restrict search area
    def margin_search(self, warped):

```

The functions of tracking lane line in video as follow:
1. `def blind_search(self, warped)` initial blind searching using slide window detection
2. `def margin_search(self, warped)` following marginal searching using fix window detection
3. `def find_lane(self, warped)` for each searching result, we perform a sanity check, if the check pass, we call marginal search function in next frame. Otherwise we call blind search search function in next frame.

Initially we use the slide window method to blind search for lane lines. The method detail described in [this video](https://www.youtube.com/watch?time_continue=25&v=siAMDK8C_x8).

Once we have a reliable lane detection and their coefficients, we use marginal search for subsequent frames, by placing windows on previous lane line position, which like search within a margin of last true detection.

There are two major challenges in correctly detecting lane lines (in a window). One is missing lane line pixels in warped image. Since no lane line pixel present, we have no way to "search" lane line. The other one is too much background noise bury all lane line pixels. These two scenario present as either too low or too high in the number of positive pixels within a window. In such case we replace them with the artificial pixels scatter around previous line position in this window.

Detection in bottom 5 windows (totally 9 windows are used) is vital since they are sufficient to estimate the polynomial. Thus if we successfully detect real pixels in all 5 bottom windows, we do not fill artificial pixels even if we fail in these windows.

![alt text][image7]
as you can see there is only one small piece of white lane line, thus we fill in a group of fake pixels (blue color) as an approximation.

![alt text][image8]
here we do not fill fake pixels in left lane line because we have a solid detect of bottom 5 windows.

Note that when identifying lane lines, I use yellow binary to find left lane line and white binary to find right lane line. This can be considered cheating here since not all road has left yellow lane and right white lane. But I think simple techniques using histogram thresholding and kurtosis over histogram can let the program identify if it is the right color on each half side.

After computing left fit and right fit, we do a simple sanity check. One is to check the base distance is within a reasonable range. Other one is to check if the difference of derivative small than a threshold, which indicates how parallel they are.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this using function `def get_deviation(self)` and `def get_curvature(self)` in `Lane()` class.

To calculate curvatures, we first obtain current fit and take the 2nd order coefficient, using it to generate a bunch of fake pixels. Then convert this pixels into meter scale and fit polynomials. Finally use polynomials in meter scale to compute curvatures. Here is a [tutorial for computing curvature](http://www.intmath.com/applications-differentiation/8-radius-curvature.php).

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code for this step is contained in the code cell under `Drawing the lines back down onto the road` of the IPython notebook.  Here is an example of my result on a test image:

![alt text][image9]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to project_video result](https://youtu.be/31Qc-FAOEow)

Here's a [link to challenges_video result](https://youtu.be/DtFSR1yusc8)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

It takes a lot of time to understand all aspect and challenge of the this project. Most of the labor devoted into tuning various parameter like thresholds. Also coming up with a robust color detection is not obvious at the beginning.

The pipeline will likely to fail in extreme light condition where lane line blends with surroundings, or background color is similar to lane line (such as yellow lane line on yellowish road surface). A robust edge detection combine with noise filter can be apply to make it see more things. Also some machine learning techniques such as bayesian inference could be applied here when the lane line is impossible to retrieve from the image.
