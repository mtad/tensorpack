
# Efficient Data Loading

This tutorial gives an overview of how to efficiently load data in tensorpack, using ImageNet
dataset as an example.

Note that the actual performance would depend on not only the disk, but also
memory (for caching) and CPU (for data processing), so the solution in this tutorial is
not necessarily the best for different scenarios.

### Use TensorFlow queues

In general, `feed_dict` is slow and should never appear in your critical loop.
i.e., you should avoid loops like this:
```python
while True:
  X, y = get_some_data()
  minimize_op.run(feed_dict={'X': X, 'y': y})
```
However, when you need to load data from Python-side, this is the only available interface in frameworks such as Keras, tflearn.

You should use something like this instead:
```python
# Thread 1:
while True:
  X, y = get_some_data()
  enqueue.run(feed_dict={'X': X, 'y': y})	 # feed data to a TensorFlow queue

# Thread 2:
while True:
  minimize_op.run()	 # minimize_op was built from dequeued tensors
```

This is now automatically handled by tensorpack trainers already (unless you used the demo ``SimpleTrainer``),
see [Trainer](trainer.md) for details.
TensorFlow is providing staging interface which may further improve the speed. This is
[issue#140](https://github.com/ppwwyyxx/tensorpack/issues/140).

You can also avoid `feed_dict` by using TensorFlow native operators to read data, which is also
supported here.
It probably allows you to reach the best performance, but at the cost of implementing the
reading / preprocessing ops in C++ if there isn't one for your task. We won't talk about it here.

### Figure out your bottleneck

For training we will only worry about the throughput but not the latency.
Thread 1 & 2 runs in parallel, and the faster one will block to wait for the slower one.
So the overall throughput will appear to be the slower one.

There isn't a way to accurately benchmark the two threads while they are running, without introducing overhead. But
there are ways to understand which one is the bottleneck:

1. Use the average occupancy (size) of the queue. This information is summarized after every epoch (TODO depend on #125).
	If the queue is nearly empty, then the data thread is the bottleneck.

2. Benchmark them separately. You can use `TestDataSpeed` to benchmark a DataFlow, and
	 use `FakeData` as a fast replacement in a dry run to benchmark the training
	 iterations.

### Load ImageNet efficiently

We take ImageNet dataset as an example of how to optimize a DataFlow.
We use ILSVRC12 training set, which contains 1.28 million images.
Following the [ResNet example](../examples/ResNet), our pre-processing need images in their original resolution, so we'll read the original
dataset instead of a down-sampled version here.
The average resolution is about 400x350 <sup>[[1]]</sup>.
The original images (JPEG compressed) are 140G in total.

We start from a simple DataFlow:
```python
from tensorpack import *
ds = dataset.ILSVRC12('/path/to/ILSVRC12', 'train', shuffle=True)
ds = BatchData(ds, 256, use_list=True)
TestDataSpeed(ds).start_test()
```

Here the first `ds` simply reads original images from filesystem, and second `ds` batch them, so
that we can test the speed of this DataFlow in the unit of batch per second. By default `BatchData`
will concatenate the data into an ndarray, but since images are originally of different shapes, we use
`use_list=True` so that it just produces lists.


[1]: #ref

<div id=ref> </div>

[[1]]. [ImageNet: A Large-Scale Hierarchical Image Database](http://www.image-net.org/papers/imagenet_cvpr09.pdf), CVPR09
