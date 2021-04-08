# **Finding Lane Lines on the Road** 

## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file. But feel free to use some other method and submit a pdf if you prefer.

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"
[image2]: ./writeup_images/image_original.png "Original"
[image3]: ./writeup_images/image_greyscale.png "Grey Scale"
[image4]: ./writeup_images/image_greyscale_blurred.png "Grey scale Blurred"
[image5]: ./writeup_images/image_cannyedges.png "Canny Edges"
[image6]: ./writeup_images/image_houghlines_extrapolated.png "Hough Lines Extrapolated"
[image7]: ./writeup_images/image_lanelines_annotated.png "Lane Lines Annotated"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

#### **Pipeline Pseudocode:**
My pipeline consists of the following overall steps:
* Grab image
    * ![alt text][image2]
* Convert to grayscale
    * ![alt text][image3]
* Apply gaussian smoothing to remove noise and facilitate the Canny algorithm
    * ![alt text][image4]
* Apply the Canny algorithm to identify high gradient sectors of the image
    * ![alt text][image5]
* Define and create a mask used to delineate an "area of interest"
* Using the Hough transform, identify line segments found within the defined "area of interest" of the image
* Line segments generated via the Hough transform are then filtered and averaged to generate a line equation for two separate line demarking the left and right lane lines
    * In the case of a video input, a circular FIFO buffer of configurable length is used to provide an average value of the line equations over the past number of frames. This averaging acts as a "low pass" filter and eliminates "jitter" in the overlaid lane lines. The size of the buffer may be configured via:
    ```python
       parameter_Hough_Extrapolated_buffer_size_video 
    ``` 
    * ![alt text][image6]
* The line equations are then converted to points which are then drawn and overlaid on the original image.
    * ![alt text][image7]   

#### **Pipeline Configurable Parameters:**
The pipeline requires the following parameters to be set initially:
```python
# Define Gaussian Blur Parameters
parameter_Gaussian_kernel_size = 9

# Define Canny Parameters
parameter_Canny_low_threshold = 50      # Low thresholds to continue edge curves 
parameter_Canny_high_threshold = 150    # High threshold to start edge curves

# Define the Hough transform parameters
parameter_Hough_rho = 1                 # Distance resolution in pixels of the Hough grid
parameter_Hough_theta = np.pi/180       # Angular resolution in radians of the Hough grid
parameter_Hough_threshold = 25          # Minimum number of votes (intersections in Hough grid cell)
parameter_Hough_min_line_length = 40    # Minimum number of pixels making up a line
parameter_Hough_max_line_gap = 100      # Maximum gap in pixels between connectable line segments

#Define Mask Parameters
parameter_Mask_offset_vert = 327
parameter_Mask_offset_hrz = 20

#Define Extrapolated Hough transform parameters
parameter_Hough_Extrapolated_buffer_size_image = 1
parameter_Hough_Extrapolated_buffer_size_video = 7 # ideal values between 5 and 10
```

#### **Pipeline Functions:**
In order to draw single lines over the left and right lane indications, a few functions were created:

```python
def equation_line(line)
    ...
    ...
   return m,b # y = mx + b
``` 
The **equation_line()** function extracts and returns the line slope and y-intercept values from the coordinates of the line provided. 

This function is imperative to the line averaging function used to average the slope and y-intercept values of the individual lines provided by the Hough transform. 

This function is called from the **extrapolate_lines_lowpass()** function.   

```python
def extrapolate_lines_lowpass(lines,mask_vert_offset,circular_buffer_size)
    ...
    ...    
    return lines_output
``` 
The **extrapolate_lines_lowpass()** function serves to calculate the average of the left and right lane line equations within the individual frame. It also, when configured via the input **circular_buffer_size()**, will average the line equations over several frames. The function returns the left and right lane line overlays in list form.  

A slope filter has been implemented to reject Hough lines with slopes exceeding user settable limits (see parameters above). The slope of the lane lines created via the parallax effect tends to be relatively consistent across the various images and videos provided in the problem set. This may be attributed to the fact that images and video were captured via a camera at a set vertical distance offset from the surface of the road as well as the path traveled being relatively level. The implementation of the filter proved effective in eliminating any "stray" lines captured by the Hough transform and provided a more accurate tracking of the lane lines. 

The implementation of a rolling average over several frames was implemented due to a fair amount of "jitter" noticed in the overlaid lines in both sample videos. The implementation of a rolling average acts as a "low pass" filter and, quite  noticeably dampens the jitter.

The following discribes the flow of operations carried out by the **extrapolate_lines_lowpass()** function:
* Loop through line array output from the Probabilistic Hough Line Transform for current frame
    * Calculate slope of line
    * Separate negative and positive slopes
    * Extract line slope - "m" and y-intercept -"b" for all detected lines
    * Filter out slope values outside of defined limits
    * Separate slope and y-intercept values dependant on positive or negative slope values
    * Store positive/negative slope and y-intercept values in list
* Ensure none of the captured slope and y-intercepts are null
    * If null: Disregard all lines captured in frame and use rolling average captured in previous x frames
    * Else: Continue
* Calculate average of all captured slope and y-intercept values in current frame
* Insert averaged slope and y-intercept values into rolling average FIFO buffer (previous x frames)
* Calculate average of left and right lane slope and y-intercept values in the past x frames
* Calculate starting and ending points for right and left lane line using bottom of frame and top of mask as y-coordinates
* Output coordinates in list form

This function is called from the **hough_lines_extrapolated()** function.

```python
def hough_lines_extrapolated(img, rho, theta, threshold, min_line_len, max_line_gap, mask_vert_offset, circular_buffer_size)
    ...
    ...
    return line_img
```
The **hough_lines_extrapolated()** function is a variation on the original **hough_lines()** function provided with the problem set. This function returns an image with the averaged and extrapolated hough lines representing the left and right lane detections.

This modified function inserts a call to **extrapolate_lines_lowpass()** between the output of the **cv2.HoughLinesP()** function and call to the **draw_lines()**. Hence the lines generated from the **hough_lines_extrapolated()** function are fed as input to the **draw_lines()** function which in-turn creates an image with the averaged and extrapolated hough lines.

This function is called from the main pipeline (for image analysis) and the  **process_image()** function for video analysis.

### 2. Identify potential shortcomings with your current pipeline

The primary shortcoming of the current pipeline may be tied directly to its lack of robustness. The parameters that must be hard coded by the user (as defined above) serve as clear indicator of the pipeline's weaknesses. 

**Canny Parameters:** The Canny parameters have been specifically tuned to detect edges of white lane indicators on a darker surface with specific weather and lighting conditions. Should these conditions change, the contrast of the lane lines against the road surface may drop significantly or even disappear. Consequently the Canny edge detection may yield null results causing the entire pipeline to fail.  

**Averaged Lines:** The pipeline is built around the assumption that the left and right lane markers may be approximated and indicated by single lines. Should the road curve by any significant degree, the line overlays will no longer accurately indicate the position of the line markers.

**Slope Limits:** Slope limits were added as parameters to weed out any stray Hough lines or anomalies occurring with the current set Hough parameters. At lower speeds with sharper radius curves, the Hough lines  may simply be rejected by the pipelines, further skewing the ability to accurately identify line markers. Also, should the camera position's vertical offset from the surface of the road be altered, the parallax effect of the lane indicators may yield a slope outside of the set limits. Significant up or down slopes in the road ahead may also cause issues for the current pipeline.    

**Mask:** The current mask parameters are hardcoded based on an input image size of 540 x 960. Should the image size change, the mask will be incorrectly defined leading the entire pipeline to utterly fail.

**Line Averaging Over Multiple Frames:** As indicated above, in order to reduced the "jitter" noticed on the line overlays generated, an averaging of the line equations over multiple frames was implemented. Although this does solve the issue of the "jitter", it also adds a "damping-like" effect. Hence, the pipeline may not accurately track the road lane indications if the vehicle itself is moving faster or a rode with closely spaced chicanes or up / down slopes.   


### 3. Suggest possible improvements to your pipeline

Addressing some of the weaknesses identified above, possible improvements to the pipeline are listed below: 

**Canny Parameters:** Modify / tune Canny parameters to work across a broader range of contrasts. This may yield more stray Hough lines and hence the checks added to the current pipeline (Slope limits, Mask) may prove useful. A balance would need to be struck such that broader Canny parameters do not yield more stringent checks, ultimately reducing the robustness of the pipeline. 

**Averaged Lines:** Currently the pipeline assumes left and right lane indications can be approximated via two lines, leading it failing when the road curves. Instead of averaging the various Hough lines, a more robust pipeline may attempt to provide a spline to overlay on the lane makings. The start and stop line points provided by the **HoughLinesP()** function  transform may be used to interpolate a piecewise spline allowing curving lane marking to be annotated.  

**Slope Limits:** Slope limits may be tuned to reject clearly stray Hough lines (slope values close to 0 or 1). However it could also be assumed that a properly tuned Canny / Hough parameters will not yield stray Hough lines and hence slope limits may not be required at all.   

**Mask:** Instead of hard coding mask vertices, these values should be parameterized based on the size of the input image. The size of the input image may be extracted via the **img.shape()** function.  

**Line Averaging Over Multiple Frames:** In order to avoid any lag in lane annotating, the amount of frames over which the the average is calculated could be made dynamic. Inputs such as the video frame rate and even the vehicle speed (if possible) could be used to dynamically adjust the size rolling FIFO buffer.