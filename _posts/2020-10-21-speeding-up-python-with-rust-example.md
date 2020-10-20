---
title: "Speeding up Python programs with Rust: a practical example"
layout: post
---

In a [previous blog post][prev] I have presented TTST, a Python program that trains itself to play Tic-Tac-Toe. Now I want to make it go faster.

# The performance of the pure Python implementation
I will present a flame chart, a graphical represention of total computation time aggregated by function calls. Reading top-down we can see all the function calls made while training, going from `main()` down to the most low-level routines.

Here TTST is training 100'000 matches against itself, with a total runtime of 44 seconds:

![ttst flamechart](/media/20201021n1e5_ttst.png)

We can notice that most of the time is spent running the function `cogitate()`:

```python
def cogitate(state, brain_map):
    weights = brain_map[state] / sum(brain_map[state])
    return np.random.choice(9, 1, p=weights)[0]
```

The culprit is `random.choice()` from Numpy, we use it to select the next move on the tic-tac-toe board: to balance [exploitation and exploration][wiki], the moves are picked at random, but the probability distribution is updated after each match to reward moves that have won games.

The variable `brain_map` is a big dictionary holding a probability distribution for each possible game state. (Illegal moves, i.e. picking a spot already taken on the board, have their probability hardcoded to zero.)

The function is quite efficient, but here the overheads are massive. I haven't looked at the Numpy internals, but it seems that there is quite a bit of setup time before the actual (fast and efficient) computation is done. And since here we are doing _many_ small iterations of `random.choice()`, all this setup time adds up.


# Rewriting the bottleneck in Rust
We will use the [`rand`][rand] library to replace the `random.choice()` function.

First let's look at a proof of concept:

```rust
use rand::prelude::*;
use rand::distributions::WeightedIndex;

fn cogitate(weights: Vec<i32>) -> i32 {
    let choices = [0,1,2,3,4,5,6,7,8];
    let dist = WeightedIndex::new(&weights).unwrap();
    let mut rng = thread_rng();

    choiches[dist.sample(&mut rng)]
}
```

Here `weights` is the equivalent of `brainmap[state]` as seen in the Python code above. Note that `WeightedIndex` does not require the weights to be normalized to 1,
for more information see the [docs on `WeightedIndex`][docs].

Now we will leverage [`rust-cpython`][rcpy] to execute the Rust function from inside the main Python program.

The `rust-cpython` library requires this kind of structure:

```rust
fn rust_to_python(_: Python, arguments: T1) -> PyResult<T2> {
    let result: T2 = do_something(arguments);
    Ok(result)
}
```
where `T1` is a Rust type that implements the `cpython::FromPyObject` trait.

So we rewrite our functions to fit. Here is the final Rust:
```rust
fn choose (_: Python, weights: Vec<i32>) -> PyResult<i32> {
    let choices = [0,1,2,3,4,5,6,7,8];
    let dist = WeightedIndex::new(&weights).unwrap();
    let mut rng = thread_rng();

    Ok(choices[dist.sample(&mut rng)])
}
```
and the final Python:
```python
from chooser import choose

def cogitate(state, brain_map):
    weights = brain_map[state]
    return choose(weights)
```


# Final benchmark
The Rust-boosted version is more than 6 times faster overall, with a total runtime of 6.45 seconds.

![ttst with rust flamechart](/media/20201021n1e5_ttst-blaze.png)

Of course the only change is in the `cogitate()` function: the new version runs just for 1.79 seconds as opposed to the original 35.6 seconds. Twenty times faster!


# Some important details
Don't forget the `rust-cpython` boilerplate:
```rust
use cpython::{Python, PyResult, py_module_initializer, py_fn};

// Here `chooser` is the name of the Python module
py_module_initializer!(chooser, |py, m| {
    m.add(py, "choose", py_fn!(py, choose(weights: Vec<i32>)))?;
    Ok(())
});
```

We also need to add this to our `Cargo.toml`:
```toml
[lib]
name = "chooser"
crate-type = ["cdylib"]

[dependencies]
rand = "0.7"

[dependencies.cpython]
version = "0.5"
features = ["extension-module"]
```

Finally we can generate `libchooser.so` by running `$ cargo build --release`. This file **must** be renamed to `chooser.so`, otherwise it won't be recognized by Python.


# Get the code
Try it out yourself, complete working code is on [GitHub][repo]!


# Further reading
* [Packaging and automated builds](https://github.com/moor84/rust-to-python)
* [EuroPython 2020: Extending Python with Rust --
Introduction and a hands-on demo of writing Python extension in Rust](https://ep2020.europython.eu/talks/6wuE8rA-extending-python-with-rust/)
* [FOSDEM 2020: Boosting Python with Rust -- The case of Mercurial](https://archive.fosdem.org/2020/schedule/event/python2020_rust/)
* The flamecharts were generated with [`cProfile`][cpro] and [`snakeviz`][snvz].


[prev]: https://enricomiccoli.com/2020/06/07/tic-tac-steel-toe-reinforcement.html
[wiki]: https://en.wikipedia.org/wiki/Multi-armed_bandit
[repo]: https://github.com/EnricoMiccoli/ttst-blaze
[rand]: https://crates.io/crates/rand
[docs]: https://docs.rs/rand/0.7.3/rand/distributions/weighted/struct.WeightedIndex.html
[rcpy]: https://crates.io/crates/cpython
[cpro]: https://docs.python.org/3.8/library/profile.html#module-cProfile
[snvz]: https://pypi.org/project/snakeviz/
