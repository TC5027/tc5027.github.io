---
layout: post
title:  "Single thread matrix multiplication"
date:   2023-10-26
math: true
---

The focus of this article is optimizing matrix multiplication for matrices of size a power of 2, represented row-wise.
As a reminder, the classic mathematical definition for matrix multiplication is : $c_{i,j} = \sum_{k}a_{i,k}b_{k,j}$

## Transpose
The first thing we can notice from the definition is that we go through the second matrix by column, but a column in memory is non-contiguous which implies a lot of cache misses. To solve this we can twist a bit the formula : $c_{i,j} = \sum_{k}a_{i,k}b_{j,k}^T$
We add extra work to perform the transposition itself but our matrix multiplication becomes much more cache friendly.

## Hilbert (finite automata)
Still we can improve cache usage, right now we navigate first $c_{0,0}$ then $c_{0,1}$ etc. If we sum up the lines we use for the first 4 values of $c_{i,j}$ we have
* $c_{0,0}$ needs $a_{0,}$ and $b_{0,}$
* $c_{0,1}$ needs $a_{0,}$ and $b_{1,}$
* $c_{0,2}$ needs $a_{0,}$ and $b_{2,}$
* $c_{0,3}$ needs $a_{0,}$ and $b_{3,}$

To sump up : 5 different lines : $b_{0,}$, $b_{1,}$, $b_{2,}$, $b_{3,}$ and $a_{0,}$

If we image another navigation for the $c_{i,j}$ which would produce as the 4 first values $c_{0,0}$, $c_{0,1}$, $c_{1,1}$ and $c_{1,0}$ we would have
* $c_{0,0}$ needs $a_{0,}$ and $b_{0,}$
* $c_{0,1}$ needs $a_{0,}$ and $b_{1,}$
* $c_{1,1}$ needs $a_{1,}$ and $b_{1,}$
* $c_{1,0}$ needs $a_{1,}$ and $b_{0,}$

To sump up : 4 different lines : $b_{0,}$, $b_{1,}$, $a_{1,}$ and $a_{0,}$

This is the idea behind Hilbert curve, it gives us an ordering in the 2d space much more cache friendly. The invert of the Hilbert transform can be computed with a finite automata (presented in the paper linked below).

Again we add extra work to compute the invert transform but the data traversal is even more cache friendly.

## SIMD
The multiplications between $a_{i,k}$ and $b_{j,k}$ being independant we could use SIMD to speed the whole matrix multiplication.

## Hilbert (Lindenmayer system)
Two consecutive integers share a common prefix regarding their binary representation. Because of it the automata used for the invert transform repeats the same initial transitions, which is a waste of time.
This [paper](https://eprints.cs.univie.ac.at/5726/1/loops.pdf) presents a Lindenmayer system taking advantage of the previously computed invert transform to compute the next one.

## Results
You can use this [tool](https://github.com/TC5027/matmul_measurements) to benchmark on your laptop. With ```cargo run --release benchmark``` (nightly)
{% highlight ruby %}
classic time : 4.141138448s
transpose time : 295.266114ms
hilbert_finite_automata time : 1.184832361s
simd time : 283.084357ms
hilber_lindenmayer time : 254.384203ms
{% endhighlight %}

Compiler seems to recognize vectorization opportunity for transpose but not for hilbert_finite_automata, it could be checked using godbolt.

For cache friendliness I use callgrind with ```valgrind --tool=callgrind --collect-atstart=no --instr-atstart=no --cache-sim=yes ./target/release/matmul-measurements {}```
{% highlight ruby %}
classic                    I   refs:      11,818,517,530
                           D1  miss rate:           50.0% (         50.0%   +      49.9%  )
transpose                  I   refs:      3,223,361,615
                           D1  miss rate:          12.1% (       12.0%   +      26.5%  )
hilbert-finite-automata    I   refs:      13,221,306,932
                           D1  miss rate:            0.8% (          0.8%   +        6.6%  )
simd                       I   refs:      3,928,826,438
                           D1  miss rate:           3.2% (        3.2%   +       5.1%  )
hilbert-lindenmayer-system I   refs:      3,772,434,940
                           D1  miss rate:           3.1% (        3.2%   +        1.5%  )
{% endhighlight %}


