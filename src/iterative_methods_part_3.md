<!-- <center> <h3 style="margin:0;padding:0;font-style:italic;"> Contributed by Daniel Fox </h3></center> -->
<center> <h1 style="margin:0;padding:1">Iterative Methods in Rust</h1></center>
<center> <h2 style="margin:0;padding:0">Reservoir Sampling </h2></center>

This is the third post in the series presenting the `Iterative Methods` Rust crate. Thanks to Daniel Vainsencher for inviting me to post. As discussed in the [earlier posts](http://daniel-vainsencher.github.io/book/iterative_methods_part_1.html), the `Iterative Methods` crate aims to expand the repertoire of iterative methods readily available in the Rust ecosystem. 

This post describes how the `Iterative Methods` crate facilitates easy reservoir sampling of a `StreamingIterator`. [Reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling) produces an up-to-date and relatively low cost random sample of a large stream of data. For example, suppose you want to maintain an up-to-date sample of \\(k\\) tweets from a twitter feed. At any moment, a reservoir sample of the tweets is equivalent to a random sample of \\(k\\) items from the portion of the stream that has been processed at that moment. The reservoir sampling algorithm accomplishes this without needing to know the total number of samples in the stream and it updates the sample to take into account new behavior in the data that may not have been present initially. Below we'll see how to use reservoir sampling for a `StreamingIterator` in Rust. Using animations we can see how the reservoir samples stay up-to-date as the data stream exhibits new behavior. 

## Outline of the Post
- The UI for Reservoir Sampling
- Visualizing the Evolving Distribution  
- Reservoir Means Approximate Stream Means 
- Exporting Data for Visualizations Using Adaptors

## The UI for Reservoir Sampling  

The UI uses adaptors to transform the behavior of a `StreamingIterator`. Suppose that `stream` is a `StreamingIterator` with items of type `T`. We adapt that iterator using `reservoir_iterable(stream, capacity, rng)`, whose fields are 1) a streaming iterator, 2) the capacity or sample size of the reservoir sample, and 3) a choice of a random number generator. The default is to set `rng` to `None`so that [`rand_pcg::Pcg64`](https://rust-random.github.io/book/intro.html) is used. You also have the option of using a seedable `rng` that might be useful for debugging or testing. Each item of the returned `StreamingIterator` is a `Vec<T>`, where the vector holds a reservoir sample.   
```rust, ignore
let stream = reservoir_iterable(stream, capacity, None);
while let Some(item) = stream.next() {
	// do stuff with each reservoir;
	// each item of the iterator is now a reservoir sample, 
	// not a single item of the original stream
}
```
<figcaption style="text-align:center;">Code Block 1</figcaption>

Let's look at an example demonstrating the utility of both the UI and reservoir sampling.

## Visualizing the Evolving Distribution 

Suppose that we have a stream of data for which the distribution changes. For example, suppose that `stream` is a `StreamingIterator` of floats for which the first quarter of the stream is generated by a normal distribution with mean \\( \mu = 0.25\\) and standard deviation \\(\sigma = 0.15\\), but that the final three-quarters of the stream is generated by a normal distribution with \\( \mu = 0.75\\), \\( \sigma = 0.15\\) &mdash;the mean jumps from \\( 0.25\\) to \\( 0.75\\). Here are histograms showing the initial and final distributions of the stream:

<figure style="align:center;border:0;">
<iframe id=iframe_embed style="border:none" src="reservoir_histograms_initial_final.html" height="500" width="800" title="Initial and Final Stream Distributions"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 1. The first 25% of items in the stream are generated by an initial normal distribution with &mu; = 0.25. The last 75% of items are generated by a normal distribution with &mu; = 0.75. The full stream is a mixture of these.  </figcaption>
</figure>

The reservoir sample starts off approximating the initial normal distribution centered at \\(\mu = 0.25\\). Gradually it shifts to approximate the distribution of the portion of the stream that has been processed. Here is an animation showing how the reservoir distribution evolves compared to the distribution of the portion of the stream that has been processed. 

<figure>
<iframe id=iframe_embed style="border:none;" src="reservoir_histogram_animation.html" height="500" width="800" title="Reservoir Distribution Approximate Stream Distribution"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 2. Reservoir samples approximate the stream distribution up to the point sampled. The initial reservoir distributions approximate the initial normal distribution; by the end of the iteration the reservoir distribution approximates the distribution of the entire stream, which, in this case, is a mix of two normal distributions. The index of the reservoir is the index for the sequence of reservoirs that are generated. After the initial reservoir is filled with the first capacity of items from the stream, each subsequent reservoir is obtained from the previous by replacing an item with a new one from the stream. The reservoir sampling algorithm randomly skips items from the stream to balance the accuracy of the reservoir sample with the efficiency of the algorithm. </figcaption>
</figure>

## Reservoir Means vs. Stream Means

While the animation in Figure 2 _visually_ demonstrates how the reservoir distribution tracks the stream distribution, we can also track the performance of the reservoir sampling by generating metrics with adaptors. For example, we can check that the mean of the reservoir sample is close to the mean of the portion of the stream that has been processed. So let's plot the reservoir mean vs. the mean of the portion of the stream that was sampled to produce the reservoir. The `Iterative Methods` crate allows you to accomplish this through adaptors as if you were using a Rust [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html). (If you only want to compute streaming means, reservoir samples are not the efficient way to do it. Rather, what follows is useful for checking that the means of the reservoirs are behaving as expected.)  

In order to know which portion of the stream has been sampled for each reservoir, we'll prepare the stream by enumerating its items with the `enumerate()` adaptor. This wraps each item of a `StreamingIterator` in a `Numbered{count, item}` struct that contains the original item and the index of the item. All of the adaptors are lazy, so the enumeration is added on the fly as the stream is processed. With the enumeration added in, for each reservoir we can find the item with the largest index, which we'll name `max_index`. We compare the reservoir mean to the mean of the stream up to and including that index. 

Here is the code that accomplishes this. The code is modular; once the data stream exists we adapt, adapt, adapt in whichever sequence is currently useful. As before `stream` is our `StreamingIterator` full of float samples from the pair of distributions as described above. The `stream` starts off with each item an `f64`; 
- After the `enumerate` adaptor, the items are `Numbered<f64>` with indices that will allow us to calculate `max_index` for each reservoir;
- After the `reservoir_iterable()` each item is a reservoir sample in a `Vec<Numbered<f64>>`;
- The `map` adaptor uses the named closure `reservoir_mean_and_max_index` to compute the mean and maximum index for each reservoir.  After the `map` adaptor each item is a `Numbered` struct containing the mean of the reservoir and the `max_index` indicating how much of the stream was sampled to obtain that reservoir. See the [source code](https://github.com/daniel-vainsencher/iterative_methods_rs/tree/main/examples) for the wheels within wheels.  
```rust, ignore
let stream = enumerate(stream);
let stream = reservoir_iterable(stream, capacity, None);
let stream = stream.map(reservoir_mean_and_max_index);
while let Some(item) = stream.next() {
	// you could do things here, but probably it is more 
	//convenient to use adaptors to accomplish your goals
}
```
<figcaption style="text-align:center;">Code Block 2</figcaption>

Now let's visuallly compare the means of the reservoirs and the means of the portions of the stream from which the reservoir sample was drawn. In the figure below, we see that, informally speaking, the mean of the reservoir does a nice job of approximating the mean of the portion of the stream that has been sampled. 
<figure>
<iframe  title="Reservoir and Stream Means;" id=iframe_embed style="border:none;" src="reservoir_means.html" height="400" width="700"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 3. The mean of the stream (up to the point processed) and the mean of the reservoir are compared. The reservoir means appear to give reasonable estimates of the mean of the portion of the stream that has been processed. The reservoir sampling algorithm randomly skips ahead before updating the reservoir in order to balance accuracy and efficiency. The skipping is visible in the variable distance between markers. The mean of the stream is only shown when the reservoir is updated. </figcaption>
</figure>

<!-- Calculate stats for the final reservoir mean over a bunch of runs. 
Comment on the number of computations peformed with res sampling compared to the full computation. The memory use for this iteration is proportional to the size of the capacity of the reservoir, relatively small compared to -->

What we see in these examples is that the idiomatic use of adaptors for a Rust `Iterator` that you know and love can be applied to more complex iterative methods, such as Reservoir Sampling: we abstract all the complexities to adaptors and closures in this example, keeping the work flow of the iterative method clear.

## Exporting Data for Visualizations Using Adaptors

How did we make the visualizations, you ask? Why, by exporting the data with more adaptors! Let's take a look at how we adapt the stream to write the data needed for the visualizations in this blog post. The data needed for all three visualizations was written to YAML files using only a single pass through the stream. Here is what we need to accomplish in order to obtain all the data needed for the visualizations:

- enumerate the stream of floats (the enumeration is needed to compute `max_index` as described above)
- write the enumerated stream to YAML (used in Figure 1)
- generate reservoir samples
- write reservoirs to YAML for the histogram animations (used in Figure 2)
- calculate the mean and `max_index` for each reservoir
- write the `(mean, max_index)` pair to YAML (used in Figure 3)

Finally, the only thing we do inside the loop is count the total number of reservoir samples that were made. This is helpful for initializing array sizes when making the visualizations. Here is the code:

```rust, ignore
// Enumerate the items in the stream; item type is now 
// Numbered<f64>{count: index, item: f64 value}
let stream = enumerate(stream);
// Write the enumerated stream to YAML as a side effect, 
// passing through the enumerated items
let stream = write_yaml_documents(stream, population_file.to_string())
    .expect("Create File and initialize yaml iter failed.");
// Convert items to reservoir samples of type Vec<Numbered<f64>>	
let stream = reservoir_iterable(stream, capacity, None);
// Write the reservoirs to YAML as a side effect
let stream = write_yaml_documents(
        stream, 
        reservoir_samples_file.to_string()
        ).expect("Create File and initialize yaml iter failed.");
// Convert items to 
// Numbered<f64>{count: max_index, item: reservoir mean} 
// using the named closure reservoir_mean_and_max_index 
let stream = stream.map(reservoir_mean_and_max_index);
// Write these new items to YAML as side effect
let mut stream = write_yaml_documents(
        stream, 
        reservoir_means_file.to_string()
        ).expect("Create File and initialize yaml iter failed.");
// num_res is used in the Python script for visualizations to 
// initialize array sizes
let mut num_res = 0;
while let Some(_item) = stream.next() {
    num_res += 1
}
```
<figcaption style="text-align:center;">Code Block 3</figcaption>

The head of the YAML file for the reservoir samples is shown below in Code Block 4. Each reservoir sample is in its own YAML document within a single file. 

```rust, ignore
---
- - 0
  - 0.07243419605614634
- - 1
  - 0.10434526397435201
- - 2
  - 0.29291775753278054
- - 3
  - 0.5312252557573507
```
<figcaption style="text-align:center;">Code Block 4. The head of the YAML file containing the reservoir samples. The items in the reservoir are (index, value) pairs. </figcaption>

The visualizations were generated from the data in the YAML files using the [Plotly](https://plotly.com/python/) module in Python. In the future we hope to switch to an entirely Rusty solution using the [Plotters](https://docs.rs/plotters/0.3.0/plotters/) crate. Currently, our Rust program runs Python scripts using `std::process::Command` and writes errors they might throw to `stderr`.

The iterator-adaptor idiom popular in Rust and other modern languages provides an ergonomic way to write code. As we build a repertoire of adaptors that implement useful iterative methods, we can easily deploy them in myriad combinations to meet our engineering needs. For example, the YAML adaptor can also be used for checkpoints or logs. So go ahead and run the examples. If you try out the `Iterative Methods` crate, please send us feedback! 

*Thanks to Daniel Vainsencher for inviting me to contribute to the **Iterative Methods** crate and for helpful feedback about this blog post and the implementation of reservoir sampling.*