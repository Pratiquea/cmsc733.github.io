---
layout: page
mathjax: true
title: Shake My Boundary
permalink: /2019/hw/hw0/
---

Table of Contents:
- [Due Date](#due)
- [Phase 1: Shake My Boundary](#pblite)
	- [Introduction](#intro)
	- [Overview](#overview)
	- [Filter Banks](#filters)
		- [Oriented DoG filters](#dogfilters) 
		- [Leung-Malik Filters](#lmfilters)
		- [Gabor Filters](#gaborfilters)
	- [Texton Map $$\mathcal{T}$$](#texton)
	- [Brightness Map $$\mathcal{B}$$](#brightness)
	- [Color Map $$\mathcal{C}$$](#color)
	- [Texture, Brightness and Color Gradients $$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$](#grad)
- [What you need to do](#problem)
  - [Problem Statement](#pro)
- [Submission Guidelines](#sub)
- [Collaboration Policy](#coll)

<a name='due'></a>
## Due Date 
11:59PM, Tuesday, February 26, 2019

<a name='pblite'></a>
## Phase 1: Shake My Boundary

<a name='intro'></a>
### Introduction
Boundary detection is an important, well-studied computer vision problem. Clearly it would be nice to have algorithms which know where one object stops and another starts. But boundary detection from a single image is fundamentally diffcult. Determining boundaries could require object-specific reasoning, arguably making the task hard.

Classical edge detection algorithms, including the Canny and Sobel baselines we will compare against, look for intensity discontinuities. The more recent pb (probability of boundary) boundary detectors significantly outperform these classical methods by considering texture and color gradients in addition to intensity. Qualitatively, much of this performance jump comes from the ability of the pb algorithm to suppress false positives that the classical methods produce in textured regions.

In this homework, you will develop a simplified version of pb, which finds boundaries by examining brightness, color, and texture information across multiple scales. The output of your algorithm will be a per-pixel probability of boundary. Several papers from Berkeley describe their algorithms and how their methods evolved over time. Their source code is also available for reference (don't use it). Here we investigate a simplified version of the recent
work from [this paper](google.com). Our simplified boundary detector will still significantly outperform the well regarded Canny edge detector. Evaluation is carried out against human annotations (ground truth) from a subset of the Berkeley Segmentation Data Set 500 (BSDS500).

<a name='overview'></a>
### Overview

The overview of the algorithm is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/hw0/Overview.PNG" width="100%">
  <div class="figcaption">
    Fig 1: Overview of the pb lite pipeline.
  </div>
</div>

<a name='filters'></a>
### Filter Banks
The first step of the pb lite boundary detection pipeline is to filter the image with a set of filter banks. We will create three different sets of filter banks for this purpose. Once we filter the image with these filters, we'll generate a texton map which depicts the texture in the image by clustering the filter responses. Let us denote each filter as $$\mathcal{F}_i$$ and texton map as $$\mathcal{T}$$. 

Let's talk a little more about filter banks now. Filtering is at the heart of building the low level features we are interested in. We will use filtering both to measure texture properties and to aggregate regional texture and brightness distributions. As we mentioned earlier, we'll implement three different sets of filters. Let's talk about each one of them next.

<a name='dogfilters'></a>
#### Oriented DoG filters
A simple but effective filter bank is a collection of oriented Derivative of Gaussian (DoG) filters. These filters can be created by convolving a simple Sobel filter and a Gaussian kernel and then rotating the result. Suppose we want $$o$$ orientations (from 0 to 360$$^\circ$$) and $$s$$ scales, we should end up with a total of $$ s \times o $$ filters. A sample filter bank of size $$2 \times 16$$ with 2 scales and 16 orientations is shown below. We expect you to read up on how these filter banks and generated and implement them. **DO NOT use any built-in or third party code for this.**


<div class="fig fighighlight">
  <img src="/assets/2019/hw0/DoGFilters.png" width="100%">
  <div class="figcaption">
    Fig 2: Oriented DoG filter bank.
  </div>
</div>

<a name='lmfilters'></a>
#### Leung-Malik Filters
The Leung-Malik filters or LM filters are a set of multi scale, multi orientation filter bank with 48 filters. It consists of first and second derivatives of Gaussians at 6 orientations and 3 scales making a total of 36; 8 Laplacian of Gaussian (LOG) filters; and 4 Gaussians. We consider two versions of the LM filter bank. In LM Small (LMS), the filters occur at basic scales $$\sigma=\{ 1, \sqrt{2}, 2, 2\sqrt{2}\}$$. The first and second derivative filters occur at the first three scales with an elongation factor of 3, i.e., ($$\sigma_x = \sigma $$ and $$\sigma_y = 3\sigma_x$$). The Gaussians occur at the four basic scales while the 8 LOG filters occur at $$\sigma$$ and $$3\sigma$$. For LM Large (LML), the filters occur at the basic scales $$ \sigma=\{\sqrt{2}, 2, 2\sqrt{2}, 4 \} $$. You need to implement both LMS and LML filter banks and **DO NOT use any built-in or third party code for this**. The filter bank is shown below. More details about these filters can be [found here](http://www.robots.ox.ac.uk/~vgg/research/texclass/filters.html). 

<div class="fig fighighlight">
  <img src="/assets/2019/hw0/LMFilters.jpg" width="100%">
  <div class="figcaption">
    Fig 3: Leung-Malik filter bank.
  </div>
</div>

<a name='gaborfilters'></a>
#### Gabor Filters
Gabor Filters are designed based on the filters in the human visual system. A gabor filter is a gaussian kernel function modulated by a sinusoidal plane wave. More details can be found on the [Wikipedia page](https://en.wikipedia.org/wiki/Gabor_filter). Implement any number of Gabor filters and **DO NOT use any built-in or third party code for this.** A sample of gabor filters is shown below.

<div class="fig fighighlight">
  <img src="/assets/2019/hw0/GaborFilters.jpg" width="100%">
  <div class="figcaption">
    Fig 3: Gabor filter bank.
  </div>
</div>


<a name='texton'></a>
### Texton Map $$\mathcal{T}$$
Filtering an input image with each element of your filter bank (you can have a lot of them from all the three filter banks you implemented) results in a vector of fillter responses centered on each pixel. For instance, if your filter bank has $$N$$ filters, you'll have $$N$$ filter responses at each pixel. A distribution of these $$N$$-dimensional filter responses could be thought of as encoding texture properties. We will simplify this
representation by replacing each $$N$$-dimensional vector with a discrete texton id. We will do this by clustering the filter responses at all pixels in the image in to $$K$$ textons using kmeans
(feel free to use Scikit learn's ``sklearn.cluster.KMeans`` function or implement your own). Each pixel is then represented by a one dimensional,
discrete cluster id instead of a vector of high-dimensional, real-valued filter responses (this process of dimensionality reduction from $$N$$ to 1 is called "Vector Quantization"). This can be represented with a single channel image with values in the range of $$[1, 2, 3, \cdots , K]$$. $$K =
64$$ seems to work well but feel free to experiment. To visualize the a texton map, you can try the ``matplotlib.pyplot.imshow`` command with proper scaling arguments.


<a name='brightness'></a>
### Brightness Map $$\mathcal{B}$$
The concept of the brightness map is simple, to capture the brightness changes in the image. Here, again we cluster the brightness values using kmeans clustering (grayscale equivalent of the color image) into a chosen number of clusters (16 clusters seems to work well, feel free to experiment). We call the clustered output as the brightness map $$\mathcal{B}$$. 

<a name='color'></a>
### Color Map $$\mathcal{C}$$
The concept of the color map is to capture the color changes or chrominance content in the image. Here, again we cluster the color values (you have 3 values per pixel if you have RGB color channels) using kmeans clustering (feel free to use alternative color spaces like YCbCr, HSV or Lab) into a chosen number of clusters (16 clusters seems to work well, feel free to experiment). We call the clustered output as the color map $$\mathcal{C}$$. Note that you can also cluster each color channel seprarately here. Feel free to experiment with different methods.

<a name='grad'></a>
### Texture, Brightness and Color Gradients $$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$
To obtain $$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$, we need to compute differences of values across different shapes and sizes. This can be achieved very efficiently by the use of Half-disc masks. 

Let us first implement these Half-disc masks. Here's an image of how these Half-disc masks look.

<div class="fig fighighlight">
  <img src="/assets/2019/hw0/HalfDiskMasks.png" width="100%">
  <div class="figcaption">
    Fig 4: Half disc masks.
  </div>
</div>

The half-disc masks are simply (pairs of) binary images of half-discs. This is very important because it will allow us to compute the chi-square distances (finally obtain values of $$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$) using a filtering operation, which is much faster than looping over each pixel neighborhood and aggregating counts for histograms. Forming these masks is quite be trivial. A sample set of masks (8 orientations, 3 scales) is shown in Fig. 4. *The filter banks and masks only need to be defined once and then they will be used on all images.*

$$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$ encode how much the texture, brightness and color distributions are changing at a pixel. We compute $$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$ by comparing the distributions in left/right half-disc pairs (opposing directions of filters at same scale, in Fig. 4, theleft/right pairs are shown one after another, these are easy to create as you have control over the angle) centered at a pixel. If the distributions are the similar, the gradient should be small. If the distributions are dissimilar, the gradient should be large. Because our half-discs span multiple scales and orientations, we will end up with a series of local gradient measurements encoding how quickly the texture or brightness
distributions are changing at different scales and angles.

We will compare texton, brightness and color distributions with the chi-square measure. The chi-square distance is a frequently used metric for comparing two histograms. $$\chi^2$$ distance between two histograms $$g$$ and $$h$$ with the same binning scheme is defined as follows

$$
\chi^2(g,h) = \frac{1}{2} \sum_{i=1}^K {\frac{g_i - h_i}^2{g_i + h_i}}
$$

here, $$K$$ indexes though the bins. Note that the numerator of this expression is simply the sum of squared difference between histogram
elements. The denominator adds a "soft" normalization to each bin so that less frequent elements still contribute to the overall distance.

To effciently compute $$\mathcal{T}_g, \mathcal{B}_g, \mathcal{C}_g$$, filtering can used to avoid nested loops over pixels. In
addition, the linear nature of the formula above can be exploited. At a single orientation and scale, we can use a particular pair of masks to aggregate the counts in a histogram via a filtering operation, and compute the chi-square distance (gradient) in one loop over the bins according to the following outline:

```
chi_sqr_dist = img*0
for i = 1:num_bins
	tmp = 1 where img is in bin i and 0 elsewhere
	g_i = convolve tmp with left_mask
	h_i convolve tmp with right_mask
	updare chi_sqr_dist
end
```

The above procedure should generate a 2D matrix of gradient values. Simply repeat this for all orientations and scales, you should end up with a 3D matrix of size $$m \times n \times N$$,
where $$(m,n)$$ are dimensions of the image and $$N$$ is the number of filters.

<a name='sub'></a>
## Submission Guidelines

<b> If your submission does not comply with the following guidelines, you'll be given ZERO credit </b>

### File tree and naming

Your submission on Canvas must be a zip file, following the naming convention **YourDirectoryID_hw1.zip**.  For example, xyz123_hw1.zip.  The file **must have the following directory structure**, based on the starter files

YourDirectoryID_hw1.zip.
 - data/. 
 - plot_eigen.m.
 - least_square.m.
 - report.pdf


### Report
For each section of the homework, explain briefly what you did, and describe any interesting problems you encountered and/or solutions you implemented.  You must include the following details in your writeup:

- Your understanding of eigenvectors and eigenvalues
- Your choice of outlier rejection technique for each dataset
- Limitation of each outliers rejection technique


As usual, your report must be full English sentences,**not** commented code. There is a word limit of 750 words and no minimum length requirement

<a name='coll'></a>
## Collaboration Policy
You are encouraged to discuss the ideas with your peers. However, the code should be your own, and should be the result of you exercising your own understanding of it. If you reference anyone else's code in writing your project, you must properly cite it in your code (in comments) and your writeup.  For the full honor code refer to the CMSC426 Fall 2018 website