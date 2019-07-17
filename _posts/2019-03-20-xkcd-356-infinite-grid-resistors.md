---
title: Solving the nerd sniper in XKCD 356
layout: post
---

Yeah, I was sniped all right by [356](https://xkcd.com/356/)... but I finally found a solution I could understand! Where did I find it? I made it myself.

If you aren't familiar with the comic: we are being challenged to find the equivalent resistance between two points that are a knight's move away on an infinite grid of $$1 \mathrm{\Omega}$$ resistors.

## Finite approximation
The infinite grid is quite awkward to handle, so the idea is to approximate the problem with a *very big* grid instead. Fortunately computers can handle *very big* just fine. Let $$n$$ be the number of nodes on the longer side of the grid:

$$ \Large \infty \simeq n $$

Wait that looks wrong...

$$ \Large \infty \simeq \Huge{N} $$

That's much better!

Here's the finite approximated circuit for $$n=5$$

![Circuit diagram](/media/diagram_n=5.svg)

Just like in the comic, we assume all resistors to be ideal with resistance $$1 \mathrm{\Omega}$$.

Now that we have laid out the mathematical framework, we can get to work on the implementation.

## Solving electrical circuits programmatically
There are many circuit simulators available and each of them has a particular application in mind, but what we need here is simple DC analysis of purely ideal circuits. A good way to do that is [nodal analysis](https://en.wikipedia.org/wiki/Nodal_analysis), a method that is easily automated.

Since we don't need a sophisticated analysis, my toy tool [`nodal.py`](https://github.com/EnricoMiccoli/nodal) is effective: actually just a subset of its functions is enough. In fact, this blog post was born as an off-shoot of that project.

Here's what `nodal.py` does in a nutshell:

0. it parses a file describing the circuit (this is called a _netlist_)
0. then it builds a system of linear equations using the algorithms prescribed by nodal analysis
0. finally it solves the system so that we can read the electrical potentials of all points in the circuit, with reference to a ground point that had been chosen earlier.

Using this software we can run our simulation: first we will build the grid made up of $$(n-1) \times n$$ nodes, then we'll attach a current generator $$i$$ to the nodes $$A$$ and $$B$$ specified by the problem. Using the results of the simulation, we will then be able to compute the voltage difference $$V$$ between said nodes, and the answer to the sniper is now just one application of Ohm's law away:

$$ \large R = \frac{V}{i} = \frac{e_{A} - e_{B}}{i} $$

And, spoiler alert,

$$ \large R \simeq 0.773 \: \mathrm{\Omega} $$

### Building the input for the simulator
To build the netlist we have to make some assumptions about the growth of the resistor grid. Effectively what we are doing is taking a sort of limit:

$$ \large R = \lim_{n \to \infty} R_n $$

as such we have to define the succession of values $$R_n$$. This isn't something specified in the problem, but it is reasonable enough to assume that the nodes $$A$$ and $$B$$ are centered on the grid. For simplicity, we limit ourselves to values of $$n$$ that are odd, so for a $$(n-1) \times n$$ grid we define the position of $$A$$ and $$B$$ as follows

$$ \large \begin{align}
c &= {\frac{n+1}{2}}\\
A &\mapsto (c, c-1)\\
B &\mapsto (c-1, c+1)\\
\end{align} $$

### Running the simulation
Futzing together the grid generator and the solver we get [this script](https://gist.github.com/EnricoMiccoli/d7e5ead0825523aa4daa5bc75e5ed62f). We can then run it with $$n$$ as an argument. 

<div class="update">
<span>Update</span> I rewrote the code, check out the performance improvements in the next blog post!<br>
<a href="{{ site.baseurl }}{% post_url 2019-07-17-updating-xkcd-356-sparse %}">
    Updating the solver for the infinite resistor grid
</a>
</div>

Here I plotted $$R_n$$ for $$n=3,9,27,51,81,157,171$$:

![Plot of simulation results](/media/results.svg)

(The point for $$n = 3$$ has been omitted for clarity.)


This problem has also been solved analytically on [StackExchange](https://physics.stackexchange.com/questions/2072/on-this-infinite-grid-of-resistors-whats-the-equivalent-resistance), finding $$ R = \frac{4}{\pi} - \frac{1}{2}$$, which I've drawn on the preceding graph with the horizontal blue line.

Since we know where the computation should converge, we can draw a log-log plot of the relative error. If we define

$$\large\begin{align}
R &= \frac{4}{\pi} - \frac{1}{2} \quad {\small(\mathrm{\Omega})}\\
E &= \frac{R_{n} - R}{R}
\end{align}$$

we get this plot:

![Plot of simulation errors](/media/03errorlog.svg)

We see we get a nice convergence. Not impressively fast, but nice enough.


