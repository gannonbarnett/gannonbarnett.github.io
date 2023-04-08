---
author: "Gannon barnett"
title: "Percentiles at Scale (in progress)"
date: "2023-04-01"
description: ""
tags: ["scaling", "percentiles", "prometheus", "data"]
hideMeta: true
ShowBreadCrumbs: false
---

## Percentiles are Useful
> The 'average' person may not exist.


Calculating the average, min, or max of a dataset can be efficient and useful. But these metrics don't tell us much about what the distribution actually looks like. The average is a made up point that may not even exist, and can be easily thrown off by outliers. Mins and maxes are useful for finding the bounds of the distribution, but they don't give any insight on the meat of the distribution. 

Percentiles are much better at encoding a distribution. A percentile always references a real data point, and most are hard to throw off with a single outlier. With a few percentiles - maybe the 10th, 50th, and 90th - we can get a pretty good idea of the shape of a distribution. But percentiles are much harder to calculate. They require sorting the entire data set, which can be expensive to do once, and near impossilbe to do on the fly. 

Luckily there are some tricks to make the impossible possible. 

## Approximating Percentiles
We need to sort the data to calculate an exact percentile. But if we establish pre-defined buckets, and count how many points are in each bucket, we can have good approximate percentiles without ever sorting the data. 

The basic plan:
1. Split the data set into ranges.
2. Count the number of points in each range.
3. Find the range that contains the percentile point.
4. (spicy) Linearly interpolate[^1] within the range to get a more precise approximation within a range.
[^1]: It's *inter*polate, not *extra*polate. Interpolate between points, extrapolate outside of points.

For example, let's take a latency distribution that we know to typically be between 1-50ms. We might define the following 'buckets' as our ranges:
```
(-Inf, 0)
[0,10)
[10, 20)
[20, 30)
[30, 40)
[40, 50)
[50, +Inf)
```

Now we can iterate through our data and increment the bucket the point falls in. We end up with:
```
 (-Inf, 0): 0 
    [0,10): 3
  [10, 20): 5
  [20, 30): 23
  [30, 40): 28
  [40, 50): 2
[50, +Inf): 1
```

The total number of points is 52. The 50th percentile is in the bucket with the 26th point. The 26th point is in the `[20, 30)` bucket, so we know the 50th percentile is in the range `[20, 30)`ms. 

But, we can do a little better. Although our dataset is only 52 points, let's imagine we have more points (if we only had 52 points, we probably just should've sorted the data and found the exact value), and a good bucket definition, so we can assume the distribution within the bucket is roughly evenly distributed (more on this later). 

The 50th percentile, or 26th point, is towards the end of the bucket; there are 8 points before the `[20, 30)` bucket begins, and 23 points within the `[20, 30)` bucket, so the 50th percentile is the 18th of 23 points within the `[20, 30)` bucket. Assuming an even distribution within the bucket, we can linearly interpolate to get a better approximation of the 50th percentile: it's 78% (18/23) through a bucket that's 10ms wide, starting at 20ms. Our approximated 50th percentile is 28ms. 

### Algorithm
Here's a more rigid defintion[^2] of the process demonstrated above. 
[^2]: Using a `bucket.limit` instead of `bucket.min` and `bucket.max` is simpler to implement. We can memoize a few of the sums required to approximate; the total count of values is the `b_count` of the `+Inf` bucket, and the starting index of the bucket is the `b_count` of the previous bucket, where buckets are sorted by `b_max`.

**Structures**

Let a bucket have the following properties:
```
bucket {
    // Minimum of range (constant)
    min

    // Maximum of range (constant)
    max

    // Index of the first point in this bucket
    start_i

    // Total number of points in this bucket
    count
}
```
And assume we have the following functions:
```
// Increment the count of the bucket that contains the value.
incorporate(value)

// Get the point index of a percentile.
index(percentile): index

// Get the bucket that contains the point index i.
bucket_containing(i): bucket
```

**Process**

Incorporate values continously:
```
for v in values:
    incorporate(v) 
```

Calculate the approximations as needed:
```
approximate(percentile):
    point_i = index(percentile)
    bucket = bucket_containing(point_index)
    bucket_range = bucket.max - bucket.min
    return bucket.min + ((point_i - bucket.start_i) / bucket_range) * bucket.max
```
### Choosing Good Buckets
It's *really* important to choose good buckets when using this approximate method. Without good buckets, the linear interpolation can severely decrease the accuracy of the approximation. This is most obviously imagined by increasing the bucket size - let's consider the same distribution as above, but make the buckets bigger.
```
 (-Inf, 0): 0 
    [0,20): 8
  [20, 50): 44

50th percentile approximation: 20 + 30 * ((26-8) 
```

### +Inf and -Inf

### Implementation Notes
