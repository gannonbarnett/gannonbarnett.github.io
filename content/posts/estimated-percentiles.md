---
author: "Gannon barnett"
title: "Percentiles at Scale (in progress)"
date: "2023-04-01"
description: ""
tags: ["scaling", "percentiles", "prometheus", "data"]
hideMeta: true
ShowBreadCrumbs: false
---
* [Approximating Percentiles](#approximating-percentiles)
* [Bad Buckets](#bad-buckets)
* [Prometheus](#prometheus)

Percentiles are super useful, but expensive to calculate. To calculate a percentile we need to find the point that is greater than n% of the set. Implementations usually sort the set, and then return the point at the correct index. There are two main costs for calculating a percentile: 
* Time: data set must be sorted (usually `O(n*log(n))`)
* Memory: data set must be stored (usually `O(n)`).

If we use the approximation method described below, we can reduce this to:
* Time: `O(n)`
* Memory: `O(C)`

These gains can be crucial for calculating percentiles of large data sets or streaming data.

## Approximating Percentiles
The trick is to approximate the sorting step. We can divide the data range into buckets, and track how many points fall into each bucket, which can be visualized as a histogram:

``` 
TODO: image of histogram
```

With the histogram, we can identify which bucket contains the percentile by finding the bucket which contains a given point index.

```
TODO: image of locating a percentile within a histogram 
```
Thus, we have our first result after `O(n)` operations and `C` memory: we know the 10th percentile of the above data set is in `TODO`. 

But, we can do better if we can assume the distribution within the bucket is uniform. With this assumption, we can linearly interpolate within the bucket to increase accuracy. In the above example, the 10th percentile is `TODO` points into the bucket, which we can interpolate as `TODO`.  

```
approx_percentile = bucket_min + (percentile_index / bucket_count) / bucket_width
```
## Defining Buckets
There are two properties of a good bucket set:
* All buckets have a near-uniform distribution
* The range of the data set is contained in the range of the buckets

If the bucket distribution isn't good, linear interpolation will be incorrect, and our method won't work very well. This is very important to understand when defining buckets. 
```
TODO: interactive impact of bad bucket definitions
```

To maintain accuracy, we must never ignore a data point. Ignoring a data point compromises our method for locating the bucket containing a point. Sometimes this is easy, but not always. Some datasets have defined ranges, but some do not. To handle the datasets that don't have defined ranges, we can add 'capping' buckets, one with a minimum of -infinity, and another with a max of +infinity. Although these buckets will maintain the accuracy of percentiles in non-capping buckets, we cannot linearly interpolate an infinite range, so percentiles which fall within these capping buckets are less accurate. A good bucket range would ensure that if capping buckets are needed, no important percentiles fall in the capping buckets.

## Prometheus
```
TODO: write
```
## Final Notes
Linear interpolating may cause more harm than good. Bad buckets can cause bad linear interpolation and lead to incorrect approximations.
