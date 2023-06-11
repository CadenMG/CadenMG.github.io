---
id: 2
title: "Optimizing Matrix Multiply with Rust"
subtitle: "An instructive intro to performance optimization with Rust"
date: "2023.06.11"
tags: "rust, performance"
---

![rusty-py](/images/rust-py.png)

### Problem
A highly instructive problem in Software Performance Engineering: [multiply](https://en.wikipedia.org/wiki/Matrix_multiplication) two square matrices of a given size. For this example, we'll say `n=2^10=1024`.

### Naive Python Implementation
Let's start with a very naive Python implementation to get a sense of the problem:
```Python
import random
from time import *

N = 1024

A = [[random.random() for _ in range(N)] for _ in range(N)]
B = [[random.random() for _ in range(N)] for _ in range(N)]
C = [[0] * N] * N

start = time()
for i in range(N):
    for j in range(N):
        for k in range(N):
            C[i][j] += A[i][k] * B[k][j]
end = time()

print(f"Time taken: {end - start}")
```

On my machine this took ~3 minutes. Likely not acceptable for most use cases.

### Naive Rust Implementation
Now let's try the exact same naive algorithm, but in Rust (using the dev/unoptimized build):
```rust
fn main() {
	let A = random_array();
	let B = random_array();
	let mut C = [[0.0; N]; N];
	
	let start = Instant::now();
	for i in 0..N {
		for j in 0..N {
			for k in 0..N {
				C[i][j] += A[i][k] * B[k][j]
			}
		}
	}
	let duration = start.elapsed();
	println!("Time taken: {:?}", duration);
}
```
This finished in 12 seconds on my machine - for a whopping 15x increase against the same Python implementation!
#### Why the speedup?
Python's versatile interpreter comes with a heavy performance cost. For each Python statement, the interpreter has to read, interpret, perform, and update the runtime state. Where as in Rust land, code is compiled directly to the machines native language, and the CPU can directly execute it without any need for a runtime.

### Modifying Loop Order
Now for something maybe not so obvious. Let's consider the following loop order change:
```Rust
fn main() {
	let A = random_array();
	let B = random_array();
	let mut C = [[0.0; N]; N];
	
	let start = Instant::now();
	for i in 0..N {
		for k in 0..N {
			for j in 0..N {
				C[i][j] += A[i][k] * B[k][j]
			}
		}
	}
	let duration = start.elapsed();
	println!("Time taken: {:?}", duration);
}
```
This code finishes in about 7 seconds for me. All I did was swap the loop order from i-j-k to i-k-j and we achieved a nearly 50% speedup! Let's look at all of the possible loop orders and their running time:
```
| Loop Order  | Running Time |
| ----------- | ------------ |
| i-j-k       | 11.8s        |
| i-k-j       | 7.6s         |
| j-i-k       | 10.1s        |
| j-k-i       | 21.8s        |
| k-i-j       | 7.7s         |
| k-j-i       | 21.4s        |
```
#### Why the speedup?
This difference in running time is caused by how well the loop order takes advantage of the CPU cache (specifically, it's [spatial locality](https://en.wikipedia.org/wiki/Locality_of_reference)) . If we examine the number of cache misses between the best and worst loop order, we find that for the worst loop order we have ~18.5M LL cache misses:
```
valgrind --tool=cachegrind ./target/debug/matrix
...
LL misses:      18,489,921  (    18,094,045 rd   +        395,876 wr)
...
```
but for the best loop order, we only have ~800k LL cache misses:
```
valgrind --tool=cachegrind ./target/debug/matrix
...
LL misses:      795,759  (       399,883 rd   +        395,876 wr)
...
```
The loop order which better takes advantage of the matrix memory layout will perform better.

### Compiler Optimization
Notice in the previous section that the binary lives in the `target/debug` directory. Which means I've been compiling the "dev" version this whole time. Let's see how well the "release" version performs:
```
cargo r --release
...
Finished release [optimized] target(s) in 0.01s
     Running `target/release/matrix`
Time taken: 170ns
```
That is *ridiculous* 🤯! It's hard to understate how amazing that is. We went from a Python prototype running on the order of minutes, to a binary running on the order of nanoseconds - mostly by just passing a single flag to the Rust compiler. This should always be the first trick to reach for. I feel confident that 170ns matrix multiplication should be sufficient for *most* use cases, so I'll stop the speed chasing here.

### What's next?
In the next post, we'll increase our matrix size from 1024 to 4096. This will require coming up with new techniques to further push the limits of our machines. 

As a take-home exercise, I'd like you to consider what's the fastest you could theoretically multiply two 1024x1024 matrices on your machine. Obviously 170ns is pretty good, but could we do even better? And if so, by how much exactly? What's the speed limit of our machines?

We'll explore this further in the next post.

[Full code on GitHub](https://github.com/CadenMG/matrix-optimizations)
