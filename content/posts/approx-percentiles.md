---
author: "Gannon barnett"
title: "Approximating Percentiles"
date: "2023-04-01"
description: ""
tags: ["data", "percentiles", "prometheus"]
hideMeta: true
ShowBreadCrumbs: false
draft: true
---
Percentiles are super useful, but expensive to calculate. To calculate a percentile we need to find the point that is greater than n% of the set. Implementations usually sort the set, and then return the point at the correct index. There are two main costs for calculating a percentile: 
* Time: sort all the data (usually `O(n*log(n))`)
* Memory: load all the data (usually `O(n)`).

If we use the approximation method described below, we can reduce this to:
* Time: `O(n)`
* Memory: `O(C)`

These gains can be crucial for calculating percentiles of large data sets or streaming data.

## Approximating Percentiles
The trick is to approximate the sorting step. We can divide the data range into buckets, and track how many points fall into each bucket, which can be visualized as a histogram. The below example shows a bimodal distribution of data between 0-50, with 10 buckets that are each 5 wide. In this case, we compress 1000 points into 10 counts.

![histogram](/approx-percentiles/histogram.png "Histogram")

With the histogram, we can identify which bucket contains the percentile by finding the bucket which contains a given point index. First we can calculate the index of the percentile by: `index = percentile * total_count`, where `total_count` is the sum of all counts. Then we can start from the lowest bucket and stop when we find the bucket that contains the index. For the above distribution, the 25th percentile is the 250th point, which falls in the fourth bucket. 

![histogram](/approx-percentiles/histogram-with-highlight.png "Histogram")

Thus, we have our first result after `O(n)` operations and `C` memory: we know the 10th percentile of the above data set is in `[15, 20)`. 

We can do better if we assume the distribution within the bucket is roughly uniform. With this assumption, we can linearly interpolate within the bucket to increase accuracy. In the above example, the 25th percentile is `6` points into the bucket which contains `167` points and starts at `15`, which we can interpolate as `15.18`.[^1] The real 25th percentile is `15.18`. This time we nailed it!   
[^1]: `15 + (6 / 167) * 5 = 15.18`

```
approx_percentile = bucket_min + (percentile_index / bucket_count) / bucket_width
```
## Defining Buckets
The success of the final step of this method -- linear approximation -- depends heavily on a good bucket set. There are two properties of a good bucket set:
* All buckets have a near-uniform distribution.
* The range of the data set is contained in the range of the buckets.

If the bucket distribution isn't good, linear interpolation will be incorrect, and our method won't work very well. This is very important to understand when defining buckets. 
```
TODO: interactive impact of bad bucket definitions
```

To maintain accuracy, we must never ignore a data point. Ignoring a data point compromises our method for locating the bucket containing a point. Sometimes this is easy, but not always. Some datasets have defined ranges, but some do not. To handle the datasets that don't have defined ranges, we can add 'capping' buckets, one with a minimum of -infinity, and another with a max of +infinity. Although these buckets will maintain the accuracy of percentiles in non-capping buckets, we cannot linearly interpolate an infinite range, so percentiles which fall within these capping buckets are less accurate. A good bucket range would ensure that if capping buckets are needed, no important percentiles fall in the capping buckets.

## Final Notes
I first learned of this method from [Prometheus's implementation](https://prometheus.io/docs/practices/histograms/#quantiles). 

Linear interpolating may cause more harm than good. Bad buckets can cause bad linear interpolation and lead to incorrect approximations. 

In implementation, it's usually simpler to just use buckets with limits instead of ranges. For example, a given bucket may have a count for all values less than 10. In this manner, the buckets are more of a CDF than a PDF. This is how Prometheus metrics are implemented.
```
TODO: PDF vs. CDF
``` 
