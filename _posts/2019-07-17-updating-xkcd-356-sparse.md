---
title: Updating the solver for the infinite resistor grid from xkcd 356
layout: post
---

With the new release of 
[nodal.py](https://github.com/EnricoMiccoli/nodal/tree/v1.0.0) 
I could greatly improve the code presented in the 
[previous blog post]({{ site.baseurl}}{% post_url 2019-03-20-xkcd-356-infinite-grid-resistors %}).
We now have a simpler script that will also run faster.

# What's new
The new release of `nodal` provides some new objects in a nicely packaged format, allowing me to rewrite the solver in a much **simpler** and maintainable form.

To make the code **faster** I started using
[sparse matrices](https://en.wikipedia.org/wiki/Sparse_matrix)
to represent the circuit. Since electrical networks are often not densely connected, with few or no components at all connecting any two given nodes, using the sparse methods from 
[scipy](https://docs.scipy.org/doc/scipy/reference/sparse.linalg.html)
results in memory and runtime savings.

In the new release of `nodal` applying this change is just a matter of setting the `sparse` flag to true when building the circuit from the netlist:

```python
netlist = nodal.Netlist("netlist.csv")
circuit = nodal.Circuit(netlist, sparse=True)
e = circuit.solve().result
```

# New results
In this table we can compare the new and old results of the simulation:

| n    | result (Ω) | old runtime (s) | new runtime (s) |
|------|------------|-----------------|-----------------|
|   21 |    0.77934 |          0.0289 |          0.1233 | 
|   51 |    0.77428 |          0.2435 |          0.4533 | 
|   71 |    0.77377 |          1.2203 |          0.7980 | 
|  101 |    0.77350 |          8.7821 |          1.5985 | 
|  171 |    0.77333 |        226.1542 |          4.5561 | 
|  201 |    0.77330 |               - |          6.5638 | 
|  501 |    0.77325 |               - |         45.3930 | 
| 1001 |    0.77324 |               - |        226.4865 | 

[tests run on Intel i7 processor, 8GB RAM]

The results of the old and new methods are equal up to rounding errors.

![Graph of time versus n](/media/comparison/comparison.svg)

The main take-away is that the new methods allowed me to run simulations of grids with $$n \geq 191$$, where the old simulation would instead crash because it ran out of memory.

# Complete updated code

<script src="https://gist.github.com/EnricoMiccoli/8b234bf89365e0d463b8a95cd5356c63.js"></script>

