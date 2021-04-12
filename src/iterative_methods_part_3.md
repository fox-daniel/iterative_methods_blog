<!--Aim for ~1500 words including code fragments.-->

<!-- Rework examples: 1) histogram visualization. 2) means using same data as histogram example, but only exporting means to Yaml. Ideally, the means would be sent to a Dash/Plotly webapp that live updates as the code runs-->

<!-- To do:
- means: motivate this as a demonstration of the effectiveness of RS; indicate that it is not the most efficient way to keep an updated mean of the stream. indicate that it requires knowing max_index.
- animation: show initial distribution in background with high transparency?
- use font so that `Iterator`s has code and non-code same size.
- Iterator vs. Iterable -> reconcile in blog and in library
- switch Numbered to have 'index' field instead of 'count'?
-->

<!-- 
- I think in the mean computation,  Iterator's `scan` (https://doc.rust-lang.org/std/iter/struct.Scan.html) might be better fit than `map`; unfortunately it does not seem to be supported by StreamingIterator yet.
	- yup. but not going to implement this now.
- The mean computation has a "wrong" computational complexity: for every single element entered into the reservoir, we scan the whole reservoir. 
	- yes. i struggled to find a better example that would not take a lot of effort and failed.
- Makes me wonder if we should generalize the reservoir to support incremental updates by allowing a function to be passed in that is called on every update, receives the new element and the element that is being removed and has a chance to update some annotation (such as the sum and latest sample number)
	- that could be good, but I would like to read up on applications first to understand what the needs are.
- More short term, at the time that you introduce the mean and max index and throughout the discussion of it until the relevant plot, reader has no idea why the max index is necessary. Perhaps plot first and then code that generates it?
	- DO: yup, I struggled with that. I am not doing anything yet because I may drop the means example.
- The paragraph after code block 3 feels a bit disconnected in style from the preceding discussion of reservoir interface, despite summarizing it.
	- DO: good point. will improve.
- I would mention somewhere that the mean itself is much cheaper to compute in a streaming way than the reservoir, but that this approach applies to other computations and mean is just an example.
	- will do if the means example remains.
- "In order to emphasize the ease and flexibility": I would replace with the more direct "How did we make these plots, you ask? why, some more generally useful adaptors, of course!" Emphasize what the adaptors are, other applications for them, and how we create much of the functionality from these reusable adaptors, with a little one-off glue code.
	- phrase changed. DO: follow through on this comment.

-->

# Iterative Methods: Reservoir Sampling

This is the third post in the series presenting the iterative_methods_in_rust crate. You may want to read the [first](http://daniel-vainsencher.github.io/book/iterative_methods_part_1.html) or [second](http://daniel-vainsencher.github.io/book/iterative_methods_part_2.html) by [Daniel Vainsencher](https://github.com/daniel-vainsencher) before reading this post.

This post describes how the Iterative Methods in Rust crate facilitates easy reservoir sampling as part of your iterative method of choice. [Reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling) produces an up-to-date and relatively low cost sample of a large stream of data of possibly unknown size. For example, suppose you want to maintain an up-to-date sample of tweets from a twitter feed in order to calculate statistics, but that including every tweet in the computations is too costly. A reservoir sample of the tweets will have a distribution that approximates the distribution of the stream (up to the point sampled), but is smaller and is updated less frequently; this allows for cheaper computations of statistics about the stream.

Here I will describe the basic idea of the algorithm behind reservoir sampling; in a later section I will provide the technical details. Say more...

## Outline of the Post
- The UI for Reservoir Sampling
- Example: Visualizing the Evolving Distribution  
- Example: Estimating the Evolving Mean 
- Example: Exporting Data for the Visualizations Using Adaptors
- An Efficient Implementation of Reservoir Sampling
- Testing the implementation

## The UI for Reservoir Sampling  

The UI uses adaptors to transform the behavior of `StreamingIterator`s. Suppose that `stream` is a `StreamingIterator` with items of type `T`. We adapt that iterator using `reservoir_iterable(stream, capacity, Rng)`, whose fields are 1) a streaming iterator, 2) the capacity or size of the reservoir sample, and 3) a choice of a random number generator. If the `Rng` is left to `None`, then the default [`rand_pcg::Pcg64`](https://rust-random.github.io/book/intro.html) is used. Each item of the returned `StreamingIterator` is a `Vec<T>`, where the vector holds a reservoir sample.   
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

## Example: Visualizing the Evolving Distribution 

Suppose that we have a stream of data for which the distribution changes. For example, suppose that `stream` is a `StreamingIterator` of floats for which the first quarter of the stream is generated by a normal distribution with mean \\( \mu = 0.25\\) and standard deviation \\(\sigma = 0.15\\), but that the final three-quarters of the stream is generated by a normal distribution with \\( \mu = 0.75\\), \\( \sigma = 0.15\\) &mdash;the mean jumps from \\( 0.25\\) to \\( 0.75\\). Here are histograms showing the initial and final distributions of the stream:

<!-- style="margin-left:2px;margin-right: 2px;" -->

<figure style="align:center;border:0;">
<iframe id=iframe_embed style="border:none" src="reservoir_histograms_initial_final.html" height="500" width="800" title="Initial and Final Stream Distributions"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 1. The first 25% of items in the stream are generated by an initial normal distribution with &mu; = 0.25. The last 75% of items are generated by a normal distribution with &mu; = 0.75. The full stream is a mixture of these.  </figcaption>
</figure>

The reservoir sample starts off approximating the initial normal distribution centered at \\(\mu = 0.25\\) but gradually shifts to approximate the total distribution, which is an uneven mixture of samples from both normal distributions. Here is an animation showing how the reservoir distribution evolves. 

<figure>
<iframe id=iframe_embed style="border:none;" src="reservoir_histogram_animation.html" height="500" width="800" title="Reservoir Distribution Approximate Stream Distribution"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 2. Reservoir samples approximate the stream distribution up to the point sampled. The initial reservoir distributions approximate the initial normal distribution; by the end of the iteration the reservoir distribution approximates the distribution of the entire stream, which, in this case, is a mix of two normal distributions.  </figcaption>
</figure>

## Example: Estimating the Evolving Mean

Building on the previous example, suppose that we want to track how the mean of the stream evolves. Reservoir sampling allows us to make cheaper computations (that is, faster and using less memory) to estimate the mean as the stream is processed. The Iterative Methods library allows us to use the kind of flexible adaptors you are used to from Rust's `Iterator`s to accomplish this. We'll plot the reservoir means vs. the true means of the portion of the stream that was sampled. In order to know which portion of the stream has been sampled for each reservoir, we'll prepare the stream by enumerating its items with the `enumerate()` adaptor. This wraps each item of a `StreamingIterator` in a `Numbered{count, item}` struct that contains the original item and the index of the item. All of the adaptors are lazy, so the enumeration is added on the fly as the stream is processed. 

Here is the code that accomplishes this. The code is modular; once the data stream exists we adapt, adapt, adapt, in whichever sequence is currently useful. Again, we suppose that `stream` is our `StreamingIterator` full of float samples from the pair of distributions as described above. The `stream` starts off with each item an `f64`; after the `reservoir_iterable()` each item is a reservoir sample in a `Vec<f64>`; and after the `map` adaptor each item is a `Numbered` struct containing the mean of the reservoir and the index indicating how much of the stream was sampled to obtain that reservoir.   
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

The `map` adaptor uses a named closure to compute the mean and maximum index for each reservoir. Since the reservoir is a `Vec<Numbered<f64>>`, we use standard Rust `Iterator` methods. We need to extract the `count` field to update the maximum index present in the reservoir and expose the `item` field that has the value of the sample so we can compute the mean.  
```rust, ignore
let reservoir_mean_and_max_index = |reservoir: &Vec<Numbered<&f64>>| -> Numbered<f64> {
    let mut max_index = 0i64;
    let mean: f64 = reservoir
        .iter()
        .map(|numbered| 
            {
            max_index = max(max_index, numbered.count);
            numbered.item.unwrap()
            })
        .sum();
    let mean = mean / (capacity as f64);
    Numbered {
        count: max_index,
        item: Some(mean),
    }
};
```
<figcaption style="text-align:center;">Code Block 3</figcaption>

The idiomatic use of adaptors for Rust `Iterators` that you know and love can be applied to more complex iterative methods, such as Reservoir Sampling: we abstract all the complexities to adaptors and closures in this example, keeping the work flow of the iterative method clear.

Now let's visuallly compare the means of the reservoirs and the means of the portions of the stream from which the reservoir sample was drawn. In the figure below, we see that, informally speaking, the mean of the reservoir does a nice job of approximating the stream. 
<figure>
<iframe  title="Reservoir and Stream Means;" id=iframe_embed style="border:none;" src="reservoir_means.html" height="400" width="700"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 3.</figcaption>
</figure>

<!-- Calculate stats for the final reservoir mean over a bunch of runs. 
Comment on the number of computations peformed with res sampling compared to the full computation. The memory use for this iteration is proportional to the size of the capacity of the reservoir, relatively small compared to -->


## Example: Exporting Data for the Visualizations Using Adaptors

How did we make the visualizations, you ask? Why, by exporting the data with more adaptors! Let's take a quick look at how we manipulated the stream to obtain the data needed for the visualizations in this blog post. The data needed for all three visualizations was written to Yaml files using only a single pass through the stream. We begin with a `stream` of floats. We `enumerate()` it so that we know the index of samples. We want to make computations on the full stream to compare it to reservoir sampling, so we adapt with `write_yaml_documents` to save the indexed stream for later. (This data is used to create the histograms of the initial and final distributions.) Wait, but we also want reservoir samples! Writing to Yaml is a side effect, it passes through the items that it was fed. So, instead of creating another copy of the data we just keep adapting. We adapt with `reservoir_iterable` to convert items from index-float pairs to vectors containing reservoir samples. The plots were too crowded initially, so I did not want to keep every reservoir sample. This is easy using the `step_by` adaptor: it only returns an item (at this point an item is a reservoir sample) every k-steps. Next, adapt to write the reservoirs to Yaml for the animation of the histograms (Figure 2). Those adaptations produce the data in Yaml files that were needed for the histogram animation and the histograms of the initial and full stream. We're not done yet, we still need to produce the means. So we used the `map` adaptor to convert items from reservoir samples to `Numbered{maximum index, reservoir mean}` (see the above discussion of the closure `reservoir_mean_and_max_index`). Then these are written to Yaml so that we can create Figure 3. Then finally, the only thing we do inside the loop is count the total number of reservoir samples that were made. 

```rust, ignore
let stream = enumerate(stream);
let stream = write_yaml_documents(stream, population_file.to_string())
    .expect("Create File and initialize yaml iter failed.");
let stream = reservoir_iterable(stream, capacity, None);
let stream = step_by(stream, 20);
let stream = write_yaml_documents(stream, reservoir_samples_file.to_string())
    .expect("Create File and initialize yaml iter failed.");
let stream = stream.map(reservoir_mean_and_max_index);
let mut stream = write_yaml_documents(stream, reservoir_means_file.to_string())
    .expect("Create File and initialize yaml iter failed.");
// num_res is used in the python script for visualizations to initialize the size of the array that will hold that data to visualize.
let mut num_res = 0;
while let Some(_item) = stream.next() {
    num_res += 1
}
```
<figcaption style="text-align:center;">Code Block 4</figcaption>



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

With mathjax we can format inline equations \\( p = \frac{log m}{log n}\\) and block equations  \\[ p = \frac{log m}{log n}\\] 