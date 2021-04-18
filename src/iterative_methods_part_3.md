<!--Aim for ~1500 words including code fragments.-->

<!-- Rework examples: 1) histogram visualization. 2) means using same data as histogram example, but only exporting means to Yaml. Ideally, the means would be sent to a Dash/Plotly webapp that live updates as the code runs-->

<!-- To do:
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
- The paragraph after code block 3 feels a bit disconnected in style from the preceding discussion of reservoir interface, despite summarizing it.
	- DO: good point. will improve.
- I would mention somewhere that the mean itself is much cheaper to compute in a streaming way than the reservoir, but that this approach applies to other computations and mean is just an example.
	- will do if the means example remains.
- "In order to emphasize the ease and flexibility": I would replace with the more direct "How did we make these plots, you ask? why, some more generally useful adaptors, of course!" Emphasize what the adaptors are, other applications for them, and how we create much of the functionality from these reusable adaptors, with a little one-off glue code.
	- phrase changed. DO: follow through on this comment.

-->

# Iterative Methods in Rust: Reservoir Sampling

This is the third post in the series presenting the `Iterative Methods` Rust crate. If you haven't already, you may want to read the [first](http://daniel-vainsencher.github.io/book/iterative_methods_part_1.html) or [second](http://daniel-vainsencher.github.io/book/iterative_methods_part_2.html) post by [Daniel Vainsencher](https://github.com/daniel-vainsencher) before continuing here. As discussed in the earlier posts, the `Iterative Methods` crate has two motivations: 1) extend the idiomatic use of iterators and adaptors in Rust to `StreamingIterator`s and 2) expand the repertoire of iterative methods readily available in the Rust ecosystem. 

This post describes how the `Iterative Methods` crate facilitates easy reservoir sampling of a `StreamingIterator`. [Reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling) produces an up-to-date and relatively low cost random sample of a large stream of data. For example, suppose you want to maintain an up-to-date sample of \\(k\\) tweets from a twitter feed. At any moment, a reservoir sample of the tweets is equivalent to a random sample of \\(k\\) items from the portion of the stream that has been processed at that moment. The reservoir sampling algorithm accomplishes this without needing to know the total number of samples in the stream and it updates the sample to take into account new behavior in the data that may not have been present initially. Below we'll see how to use reservoir sampling for `StreamingIterator`s in Rust. Using animations we can see how the reservoir samples stay up-to-date as the data stream exhibits new behavior. 

## Outline of the Post
- The UI for Reservoir Sampling
- Example: Visualizing the Evolving Distribution  
- Example: Reservoir vs. Stream Means 
- Example: Exporting Data for the Visualizations Using Adaptors
- An Efficient Implementation of Reservoir Sampling

## The UI for Reservoir Sampling  

The UI uses adaptors to transform the behavior of `StreamingIterator`s. Suppose that `stream` is a `StreamingIterator` with items of type `T`. We adapt that iterator using `reservoir_iterable(stream, capacity, rng)`, whose fields are 1) a streaming iterator, 2) the capacity or size of the reservoir sample, and 3) a choice of a random number generator. If the `rng` is left to `None`, then the default [`rand_pcg::Pcg64`](https://rust-random.github.io/book/intro.html) is used. Each item of the returned `StreamingIterator` is a `Vec<T>`, where the vector holds a reservoir sample.   
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

The reservoir sample starts off approximating the initial normal distribution centered at \\(\mu = 0.25\\) but gradually shifts to approximate the distribution of the portion of the stream that has been processed. Here is an animation showing how the reservoir distribution evolves compared to the distribution of the portion of the stream that has been processed. 

<figure>
<iframe id=iframe_embed style="border:none;" src="reservoir_histogram_animation.html" height="500" width="800" title="Reservoir Distribution Approximate Stream Distribution"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 2. Reservoir samples approximate the stream distribution up to the point sampled. The initial reservoir distributions approximate the initial normal distribution; by the end of the iteration the reservoir distribution approximates the distribution of the entire stream, which, in this case, is a mix of two normal distributions. The index of the reservoir is the index for the sequence of reservoirs that are generated. After the initial reservoir is filled with the first capacity of items from the stream, each subsequent reservoir is obtained from the previous by replacing an item with a new one from the stream. The reservoir sampling algorithm randomly skips items from the stream to balance the accuracy of the reservoir sample with the efficiency of the algorithm. </figcaption>
</figure>

## Example: Reservoir vs. Stream Means

The animation in Figure 2 allows us to see how the reservoir distribution tracks the stream distribution. We can also check that the mean of the reservoir is close to the mean of the portion of the stream that has been processed. Now, if you want to compute streaming means, what follows is not the efficient way to do it. Rather, what follows is useful for checking that the means of the reservoirs are behaving as expected. The `Iterative Methods` crate allows us to use the kind of flexible adaptors you are used to from Rust's `Iterator`s to accomplish this. We'll plot the reservoir mean vs. the mean of the portion of the stream that was sampled to produce the reservoir. 

In order to know which portion of the stream has been sampled for each reservoir, we'll prepare the stream by enumerating its items with the `enumerate()` adaptor. This wraps each item of a `StreamingIterator` in a `Numbered{count, item}` struct that contains the original item and the index of the item. All of the adaptors are lazy, so the enumeration is added on the fly as the stream is processed. With the enumeration added in, for each reservoir we can find the item with the largest index, which we'll name `max_index`. We compare the reservoir mean to the mean of the stream up to and including that index. 

Here is the code that accomplishes this. The code is modular; once the data stream exists we adapt, adapt, adapt in whichever sequence is currently useful. Again, we suppose that `stream` is our `StreamingIterator` full of float samples from the pair of distributions as described above. The `stream` starts off with each item an `f64`; 
- after the `enumerate` adaptor, the items are `Numbered<f64>` with indices that will allow us to calculate `max_index` for each reservoir
- after the `reservoir_iterable()` each item is a reservoir sample in a `Vec<Numbered<f64>>`
- and after the `map` adaptor each item is a `Numbered` struct containing the mean of the reservoir and the `max_index` indicating how much of the stream was sampled to obtain that reservoir.   
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

The `map` adaptor uses a named closure to compute the mean and maximum index for each reservoir. Since the reservoir is a `Vec<Numbered<f64>>`, we use standard Rust `Iterator` methods. We need to extract the `count` field to update the maximum index present in the reservoir and expose the `item` field that has the value of the sample so we can compute the mean. Applying the `.fold()` adaptor with the accumulator a tuple `(max_index, sum_of_values)` reduces the reservoir down to the pair needed, except that we still have to divide the sum by the capacity to obtain the mean.  
```rust, ignore
let reservoir_mean_and_max_index = |reservoir: &Vec<Numbered<f64>>| -> Numbered<f64> {
        let result = reservoir.iter().fold((0, 0.), |acc, x| {
            (cmp::max(acc.0, x.count), acc.1 + x.item.unwrap())
        });
        let max_index = result.0;
        let mean = result.1 / (capacity as f64);
        Numbered {
            count: max_index,
            item: Some(mean),
        }
    };
```
<figcaption style="text-align:center;">Code Block 3</figcaption>

Now let's visuallly compare the means of the reservoirs and the means of the portions of the stream from which the reservoir sample was drawn. In the figure below, we see that, informally speaking, the mean of the reservoir does a nice job of approximating the mean of the portion of the stream that has been sampled. 
<figure>
<iframe  title="Reservoir and Stream Means;" id=iframe_embed style="border:none;" src="reservoir_means.html" height="400" width="700"> </iframe>
<figcaption style="text-align:center;font-size:14px">Figure 3. The mean of the stream (up to the point processed) and the mean of the reservoir are compared. The reservoir means appear to give reasonable estimates of the mean of the portion of the stream that has been processed. The reservoir sampling algorithm randomly skips ahead before updating the reservoir in order to balance accuracy and efficiency. The skipping is visible in the variable distance between markers. The mean of the stream is only shown when the reservoir is updated. </figcaption>
</figure>

<!-- Calculate stats for the final reservoir mean over a bunch of runs. 
Comment on the number of computations peformed with res sampling compared to the full computation. The memory use for this iteration is proportional to the size of the capacity of the reservoir, relatively small compared to -->

What we see in these examples is that the idiomatic use of adaptors for Rust `Iterators` that you know and love can be applied to more complex iterative methods, such as Reservoir Sampling: we abstract all the complexities to adaptors and closures in this example, keeping the work flow of the iterative method clear.

## Example: Exporting Data for the Visualizations Using Adaptors

How did we make the visualizations, you ask? Why, by exporting the data with more adaptors! Let's take a quick look at how we manipulated the stream to obtain the data needed for the visualizations in this blog post. The data needed for all three visualizations was written to Yaml files using only a single pass through the stream. We begin with a `stream` of floats. We `enumerate()` it so that we know the index of samples. We want to make computations on the full stream to compare it to reservoir sampling, so we adapt with `write_yaml_documents` to save the indexed stream for later. (This data is used to create the histograms of the initial and final distributions in Figure 1.) Wait, but we also want reservoir samples! Writing to Yaml is a side effect, it passes through the items that it was fed. So we keep adapting. We adapt with `reservoir_iterable` to convert items from index-float pairs to vectors containing reservoir samples (where the items in the sample are still index-float pairs). Next, adapt to write the reservoirs to Yaml for the animation of the histograms (Figure 2). We're not done yet, we still need to produce the means. So we use the `map` adaptor to convert items from reservoir samples to `Numbered{maximum index, reservoir mean}` (see the above discussion of the closure `reservoir_mean_and_max_index`). These are written to Yaml so that we can create Figure 3. Finally, the only thing we do inside the loop is count the total number of reservoir samples that were made. This is used to make the visualizations. Here is the code:
```rust, ignore
let stream = enumerate(stream);
let stream = write_yaml_documents(stream, population_file.to_string())
    .expect("Create File and initialize yaml iter failed.");
let stream = reservoir_iterable(stream, capacity, None);
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

The visualizations were generated from the data in the Yaml files using the [Plotly](https://plotly.com/python/) module in Python. In the future we hope to switch to an entirely Rusty solution using the [Plotters](https://docs.rs/plotters/0.3.0/plotters/) crate.

## An Efficient Implementation of Reservoir Sampling

The final topic we'll cover is the implementation of reservoir sampling. The `Iterative Methods` crate uses [*Algorithm L*](https://en.wikipedia.org/wiki/Reservoir_sampling#An_optimal_algorithm), an optimal algorithm introduced by Kim-Hung Li. The basic idea of the algorithm is to not update the reservoir with every new item of the stream that is processed, but to randomly skip ahead before processing a new item. The cleverness resides in choosing *how much* to skip ahead. See the [research article]((https://dl.acm.org/doi/10.1145/198429.198435)) by Kim-Hung Li for the details. 

To implement the algorithm in the library we want an adaptor `reservoir_iterable()` that can be applied to any `StreamingIterator`, `I`. The adaptor wraps `I` into a new struct that maintains state:
```rust, ignore
#[derive(Debug, Clone)]
pub struct ReservoirIterable<I, T> {
    it: I,
    pub reservoir: Vec<T>,
    capacity: usize,
    w: f64,
    skip: usize,
    rng: Pcg64,
}
```
The state includes the underlying `I`, the reservoir sample which is a `Vec<T>`, and the capacity of the reservoir. Before discussing the other variables, let's note that `reservoir` is public because this part of state constitutes the items of the new `StreamingIterator` that are produced. The remaining parts of state, which are for internal use only, are a weight `w`, a skip size `skip`, and a choice of a random number generator `rng`. The rng can be selected when applying the adaptor or set as `None` to use the default. Selecting the rng can be important. For example, for running tests you may want to use a seeded rng so that behavior is reproducible and aids debugging. To understand the weight and skip size, let's look at the implementation of the `advance` method that is required of all `StreamingIterator`s.

To create the first reservoir, we simply fill it with the first `capacity` elements. That is accomplished in the first arm of the conditional `if` statement: until the reservoir is full to capacity we add each subsequent item of the stream to it. The items added are `clone()`s of the items from the stream. 

The more interesting part of the algorithm begins once the reservoir is full to capacity. Now we can see how `skip` and `w` are used. The variable `skip` tells the algorithm how far to skip ahead in the stream before selecting an item to add to the reservoir. We accomplish this using the adaptor `.nth()` that exists for both Rust's standard `Iterator`s and the `StreamingIterator`s we are working with here. Once we have selected an item to incorporate in the reservoir sample, we need to figure out where to put it. This is done by uniformly, randomly selecting an item in the reservoir which will be replaced. Then we need to update state to know how far to skip ahead next time. We update `w` according to the formulation of the algorithm: generate a random number `r` in \\((0,1)\\) and set \\( w = w \cdot r^{\frac{1}{k}}\\). (In the code, the kth root is taken in a more numerically stable way using logarithms.) This new weight is then used to determine the `skip`: let `r'` be another random number in \\((0,1)\\) and set \\(skip = skip + 1 + \lfloor \frac{\log(r')}{\log(1-w)}  \rfloor\\). 

```rust, ignore
fn advance(&mut self) {
    if self.reservoir.len() < self.capacity {
        while self.reservoir.len() < self.capacity {
            if let Some(datum) = self.it.next() {
                let cloned_datum = datum.clone();
                self.reservoir.push(cloned_datum);
            } else {
                break;
            }
        }
    } else if let Some(datum) = self.it.nth(self.skip) {
        let h = self.rng.gen_range(0..self.capacity) as usize;
        let datum_struct = datum.clone();
        self.reservoir[h] = datum_struct;
        self.w *= (self.rng.gen::<f64>().ln() / (self.capacity as f64)).exp();
        self.skip = ((self.rng.gen::<f64>() as f64).ln() / (1. - self.w).ln()).floor() as usize;
    }
}
```

The iterator-adaptor idiom popular in Rust and other modern languages provides an ergonomic way to write code. As we build a repertoire of adaptors that implement useful iterative methods, we can easily deploy them in myriad combinations to meet our engineering needs. If you try out the `Iterative Methods` crate, please send us feedback! 

*Thanks to Daniel Vainsencher for inviting me to contribute to the **Iterative Methods** crate and for helpful feedback about this blog post and the implementation of reservoir sampling.*
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