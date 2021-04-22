<!--Aim for ~1500 words including code fragments.-->

<!-- 
DV: Again I'd consider putting small technical comments inside the code, and leaving the text and equations for the conceptual parts. For completeness of the picture, consider adding the constructor function as well, showing us the initiialization of skip etc.
-->

# Iterative Methods in Rust: An Optimal Implementation of Reservoir Sampling

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

*Thanks to Daniel Vainsencher for inviting me to contribute to the **Iterative Methods** crate and for helpful feedback about this blog post and the implementation of reservoir sampling.*

