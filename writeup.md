# **Finding Lane Lines on the Road** 

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)
[image1]: ./examples/area_of_interest.jpg "Area of interest"
[image2]: ./examples/cropped.jpg "Image cropped to Area of interest"
[image3]: ./examples/distance_map.jpg "Distance map (inversed)"
[image4]: ./examples/edges.jpg "Canny Edges with different kernel sizes"
[image5]: ./examples/hough_lines.jpg "Hough lines"
[image6]: ./examples/groups.jpg "Line clusters"
[image7]: ./examples/trapezoid.jpg "Lane line selection"
[image8]: ./examples/result.jpg "The detected lanes"
[image9]: ./examples/heavy_noise.jpg "Canny Failure"
[image10]: ./examples/text_interference.jpg "Misleading Hough lines caused by text"
[image11]: ./examples/crossing_lanes.jpg "Trapezoid Failure"
[image12]: ./examples/perspective.jpg "Perspective mapping"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline consisted of 6 steps: 

#### 1.1 Area of interest
First, I've defined an area of interest, represented by a perspectively distorted rectangle on what I perceived as ground plane in all of the provided videos and images:

![Area of interest][image1]

The area serves as a rough filter which supresses eventually misleading line detections outside the area where we'd expect lane markings. Also the perspective of the camera causes lane lines to get thinner and more blurry as they approach the horizon, thus gradually diminishing the precision of line detection. This is why the area of interest restricts lane detection to the lower 3rd of the image/frame.

Furthermore, the area of interest improves the performance for the subsequent pipeline steps, as it allows the image to be cropped to the area bounds:

![Image cropped to Area of interest][image2]


#### 1.2 Color distance map

One of the most significant features of the lane lines we want to detect is their color. If we'd simply convert the image to grayscale, part of that important feature would get lost at the very beginning of our pipeline. That is why I've replaced the grayscale conversion with a distance map conversion:

For each pixel in the image, the conversion calculates the minimal euclidian distance between the pixels color vector (r,g ,b) and a given array of target color vectors. As target colors I have used White (255, 255, 2255) and Federal Color No. 13538 (245, 171, 48). 

Another details of the conversion is, that distances which excess a specific treshold are clamped. Later on, this helps very much to supress misleading edges in between colors we don't care for. The threshold I have decided for is 150.

The biggest challenge in this step has been to get the algorithm run fast (my first approach took like 4s to compute for one target image). Lucky, I've detected the method 'scipy.spatial.distance.cdist' which allows to calculate euclidian vector distances for multiple numpy arrays on the gpu without involving python code. That has shrunk the final computation time down to a few ms.

Since the clamping removes all gray values < 150, I've added a normalization step afterwards, based on the pixel values inside the area of interest. After the normalization, the maximum gray scale range between 0 and 255 is used again to provide an optimal basis for canny edges. I've also inverted the gray scale values (just for visuals, so all lane markings are displayed white):

![Distance map (inversed)][image3]


#### 1.3 Canny Edge Detection

In the next step, I run a Canny Edge Detection as learned in the videos. In order to remove edges between remaining unwanted details/noise, I invoke a Gaussian Blur step beforehand. Intuitively, I've chosen the blur to use the same kernel size as the aperture size of the canny edges. I've chosen the value 7 for it because it is the smallest value that eradicated all unwanted details in the examples provided. Because of the previous normalization, I can afford to set the upper threshold of Canny to a relatively high value of 150, but I keep the lower threshold at 50 in order to avoid disconnected edge pixels which could be troublesome for the upcoming Hough Lines Transformation.

![Canny Edges with different kernel sizes][image4]
 

#### 1.4 Hough Lines

In the next step, I run the probabilistic Hough Lines Transform as learned in the course. The parameters I have chosen are:
* 2 for the pixel resolution 
* pi / 90 for the angle resolution
* 10 for the line accumulation threshold
* 20 for the mininum length of a lines candidate
* 15 for maximum allowed gap between pixels of a line candidate

The reason I have chosen relatively small values for minimum line length and decreased pixel/angle resolution is so I don't miss out on line hints at broken lane markings or markings covered by tree shadows. Also the number of lines describing the same straight line will be part of the subsequent clustering heuristic so having a high number of short lines is desired.

Within the area of interest I can reglect the influnce from curves, so can I expect lane lines to vaguely point into the direction of the vanishing point. The direction of the horizontal lines is not influenced by the perspective distortion, so I can safely rule them out as candidates for lane lines.  This is done in an additional filter step at the end of Hough Lines. 

![Hough lines][image5]

#### 1.5 Line clustering

In the next step, I am trying to cluster the collected Hough Lines. This is implemented by picking an arbitrary line segment from the list of Hough lines and comparing it to the existing clusters. If there is no cluster existing yet or the similarity is worse than a given threshold, the line segment is added to a new cluster. If the line falls into the similarity threshold of one or more clusters, it is added to the cluster with the biggest similarity.

Each cluster represents a single straight line in general form Ax + Bx + C = 0. When a cluster is created, the its straight line is extrapolated from the initial line segment. If additional lines are added to the cluster, its straight line reppresents the mean straight between all added lines. I have chosen the general form for the straight line equasion over y = mx + b, because it easily allows to calculate the line distance and makes no problems with vertical lines.

The similarity function between line segment (p1, p2) and a cluster is simply defined by the sum of
a) The distance of point (p1 + p2) / 2 to the clusters straight line 
b) The weighted angle difference between the line segment and the straight line

Every lane marking is expected to have a physical extent and thus must contain 2 outer lines. So at the end of the clustering, I can safely remove all clusters with less than 2 lines.

![Line clusters][image6]

#### 1.6 Lane selection

In the next step, I have to select the actual lane markings from the calculated line clusters.
This is a very hard task, because neither the absolute number of lines inside a cluster nor accumulated length of lines inside a cluster seems like a secure indication for whether a cluster really corresponds to a lane marking.

So I decided to take the perspective implications into accout and select the two clusters with the highest combined number of lines, which form a proper trapezoid:

![Lane line selection][image7]

The detected lane lines are further run through a depth-pass filter over successive selections in the videos. This is more for visual appeasement than it serves any real purpose. Here is how the result looks:

![The detected lanes][image8]

### 2. Identify potential shortcomings with your current pipeline

There are some weak points in the steps of the pipeline.

#### 2.1 Area of interest

The area of interest implicitly assumes an appropriate direction and position. So it can potentially cut away wanted information in situations where this is not given, e.g.
* Camera center to low
* 90° side way mobile cam video

Also it has the capability to provide unwanted artifacts at its borders (e.g. neighbouring lanes parallel to the polygon lines, which happen to be sometimes just within the area, sometimes not).

#### 2.2 Color distance map

Unexpected light and or weather conditions may have a strong impact on the rgb values of the image pixels. Actual lane marking pixel might get lost under the threshold. This was actually one of the problems while dealing with the very light bridge section in the challenge video.

#### 2.3 Canny  Edge Detection

Another weak point of the algorithm is definitely the Canny Edges. It generally delivers a significant number of edges which are irrelevant for lane detection and which have to be statistically cleaned afterwards. It is specifically vulnarable against strong noise, e.g. see the following image:

![Canny Failure][image9]

#### 2.4 Hough Lines

I was forced to chose a relatively small line length for the Hough Lines in order to be still able to detect broken lane markings and curved lane markings. Both are cases in which the Hough Line algorithm seems to be the non-optimal approach. Also with the low line length threshold, now the Hough Lines step produces alot of misleading line segments when text or vehicles interfere:

![Misleading Hough lines caused by text][image10]

#### 2.5 Line clustering

The line clustering is dependant on the order in which the single line segments have been fed into the algorithm. The clustering by mean straight line only works really well if both lane marker borders are being represented by about the same number of lines. Otherwise, the mean line will not pass through the center of the lane markings afterwards. 

#### 2.6 Lane selection

The pure number of line segments in a cluster is not a secure indication for a lane marking. E.g. safety barriers along the road usually tend to be light metallic and feature clusters with an abundance of Hough Lines. 

Also in case the vanishing point is very high, lane lines might actually cross within the area of interest, in which case the correct lane line combination would be forbidden by the trapezoid restriction: 

![Trapezoid Failure][image11]


### 3. Suggest possible improvements to your pipeline

#### 3.1 An alternative algorithm

There are some given features that we do not use for the algorithm yet.

In general we can say, the information close to the camera  is quite detailed and good to interprete, then it gets harder to interpret correctly, the further we approach the vanishing point. This calls for some sort of expansion algorithm.

I would keep the distance mapping at the begin of the pipeline, it seems like an overall good preprocessing. In the second step one could add a perspective mapping, so that instead of our lane lines being pointed towards a vanishing point, they'd be somewhat parallel:

![Perspective mapping][image12]

Now, instead of the Canny/Hough combo, we could start at the bottom of the mapped image and search for lane signatures in each row of the mapped image. That could also make use of the yet unused fact, that in between the two outer borders of the physical lane marking, there is only one specific color prominent (white).

During the search upwards, we could access the lane information we've extracted from previous rows so far helping us with the decision. And so we could expand the detected lane marking pixel by pixel, following their estimated centers. This sort of expansion could follow curves and end dynamically at the point where the detection precision for the lane marking in question would fall under a specific threshold. 

In the end we would calculate a line or curve out of the gathered line centers and run a selection algorithm on the dected curves/lines similar like before.

#### 3.2 Improve lane clustering

Right now the clustering doesn't take into account that we are actually searching for two parallel edge lines which mark the borders of a lane marking. 
So instead of running a single clustering step with medium similarity threshold, there could be two hierarchical steps.
* Step one could have a very tight similarity threshold trying to cluster only one of the outer borders of each lane marking.
* Step 2 could try to pair up the clusters from step one trying two find two parallel clusters with a distance that about matches the expected width of a lane marking.

