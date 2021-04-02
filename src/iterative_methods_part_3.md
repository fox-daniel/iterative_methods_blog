# Iterative Methods: Reservoir Sampling

This is the third post in the series on iterative methods in Rust. You may want to read the [first](http://daniel-vainsencher.github.io/book/iterative_methods_part_1.html) or [second](http://daniel-vainsencher.github.io/book/iterative_methods_part_2.html) before reading this post.

Reservoir sampling allows for an up-to-date and relatively low cost sample of a large stream of data of possibly unknown size. For example, suppose you want to maintain an up to date sample of tweets from a twitter feed in order to calculate statistics, but crunching the stastics on all of the streaming tweets might be too computationally costly. Reservoir sampling is a way of maintaining a sample whose distribution mirrors that of the stream (up to the point sampled).

Here is some code I typed in here:
```rust, ignore
let iter = reservoir_iterator(iter);
let iter = enumerate(iter);
```

Here is some code from a file:
```rust, ignore
{{#include res_sampling_example.rs:28:30}}
```

New content appears when pushed to origin?

For inline equations such as \\( p = \frac{log m}{log n}\\) and block equations such as \\[ p = \frac{log m}{log n}\\]