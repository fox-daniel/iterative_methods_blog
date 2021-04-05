<!--Aim for ~1500 words including code fragments.-->
# Iterative Methods: Reservoir Sampling

This is the third post in the series presenting the iterative_methods_in_rust crate. You may want to read the [first](http://daniel-vainsencher.github.io/book/iterative_methods_part_1.html) or [second](http://daniel-vainsencher.github.io/book/iterative_methods_part_2.html) by [Daniel Vainsencher](https://github.com/daniel-vainsencher) before reading this post.

This post describes how the Iterative Methods in Rust crate facilitates easy reservoir sampling as part of your iterative method of choice. [Reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling) produces an up-to-date and relatively low cost sample of a large stream of data of possibly unknown size. For example, suppose you want to maintain an up-to-date sample of tweets from a twitter feed in order to calculate statistics, but that including every tweet in the computations is too costly. A reservoir sample of the tweets will have a distribution that approximates the distribution of the stream (up to the point sampled) but, is smaller and is updated less frequently; this allows for cheaper computations of statistics about the stream.

Here I will describe the basic idea of the algorithm behind reservoir sampling; in a later section I will provide the technical details. Say more...

## Outline of the Post
- The UI for reservoir samples
- Examples on toy data demonstrating some of the properties of reservoir sampling
	- Approximating the mean of the full data stream
	- Approximating the distribution
- The algorithmic implementation
- Testing the implementation 

## The UI for Reservoir samples using the `Iterative Methods in Rust` crate 

Let's create a stream of data of the form `[0,..,0,1,..,1]` with 500 0's and 500 1's. Let's work with reservoir sampling using a capacity of 50. Because the first 50 items in the stream are all 0, our initial reservoir is just 50 0's. As the stream is processed, we'll occassionaly update the reservoir sample. Eventually we'll start processing 1's and the will begin to enter the reservoir sample. The ratio of 1's to 0's in our reservoir should be close to the ratio of 1's to 0's that have been processed so far. Therefore, at the end of the whole stream we expect to have roughly 25 0's and 25 1's. 
```rust, ignore
let initial_iter = iter::repeat(initial_value).take(capacity);
if capacity > stream_length {
    panic!("Capacity must be less than or equal to stream length.");
}
let final_iter = iter::repeat(final_value).take(stream_length - capacity);
let stream = initial_iter.chain(final_iter);
let stream = convert(stream);
let stream = enumerate(stream);
let res_iter = reservoir_iterable(stream, capacity, None);
```

## Two Examples

TOPIC_SENTENCE. To demonstrate how to use reservoir sampling, we'll work with synthetic data that helps us see what reservoir sampling does.  

### The mean

### Visualizing Reservoir and Stream Histogram

![The mean of the reservoir tracks the mean of the stream in the following figure.](reservoir_and_stream_means.png "Reservoir and Stream Means")

Here are the initial and final distributions of the stream where the image is embedded using an iframe:

<iframe id=iframe_embed allowtransparency="true" style="border:none; background-color: #000000;" src="reservoir_histograms_initial_final.html" height="600" width="900" title="Initial and Final Stream Distributions"> </iframe>

Here is an animation showing how the reservoir distribution evolves. It tracks with stream distribution as that evolves, eventually approximating the distribution of the entire stream.

<iframe id=iframe_embed style="border:none;" src="reservoir_histogram_animation.html" height="600" width="900" title="Reservoir Distribution Approximate Stream Distribution"> </iframe>


<!-- 

Here is some code I typed into the md file:
```rust, ignore
let iter = reservoir_iterator(iter);
let iter = enumerate(iter);
```

Here is some code referenced from a file:
```rust, ignore
{{#include res_sampling_example.rs:28:30}}
```

New content appears when pushed to origin?

With mathjax we can format inline equations \\( p = \frac{log m}{log n}\\) and block equations  \\[ p = \frac{log m}{log n}\\] -->