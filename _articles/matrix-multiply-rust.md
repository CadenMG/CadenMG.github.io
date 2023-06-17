---
id: 2
title: "Optimizing Matrix Multiply with Rust"
subtitle: "An instructive intro to performance optimization with Rust"
date: "2023.06.17"
tags: "rust, performance"
---

![rusty-py](/images/rust-py.png)

### Problem
A highly instructive problem in Software Performance Engineering: [multiply](https://en.wikipedia.org/wiki/Matrix_multiplication) two square matrices of a given size.

### Naive Python Implementation
For now we'll say `N = 1024`. Let's start with a very naive Python implementation to get a sense of the problem:
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
This difference in running time is caused by how well the loop order takes advantage of the CPU cache (specifically, it's [spatial locality](https://en.wikipedia.org/wiki/Locality_of_reference)). If we examine the number of cache misses between the best and worst loop order, we find that for the worst loop order we have ~18.5M LL cache misses:
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
Time taken: 328.815344ms
```
That is *incredible* 🤯! It's hard to understate how amazing that is. We went from a Python prototype running on the order of minutes, to a binary running on the order of milliseconds - mostly by just passing a single flag to the Rust compiler. This should always be the first trick to reach for. Let's increase the matrix size to further push our machines.

### 1024 -> 4096
As promised previously, lets increase the matrix size to 4096 and look at how our algorithm performs now:
```
cargo r --release
   Compiling matrix v0.1.0 (/home/caden/matrix-optimizations/matrix)
    Finished release [optimized] target(s) in 0.24s
     Running `target/release/matrix`
Time taken: 28.704995681s
```
Ouch - that's much, much worse.

### Parallel Loops
Okay, what's everyone's first instinct when wanting to optimize something? Parallelize the work! Why only utilize a single core of the machine when we have multiple?
Where could we apply parallelization? A very natural candidate would be the loops:
```Rust
for i in 0..N {
	for k in 0..N {
		for j in 0..N {
			C[i][j] += A[i][k] * B[k][j]
		}
	}
}
```
#### Parallel loops in Rust?
Introducing [rayon](https://docs.rs/rayon/latest/rayon/) - a highly regarded data-parallelism crate for converting sequential computations into parallel ones. Adding `rayon` to my `Cargo.toml`:
```
[dependencies]
rayon = "1.7.0"
```

#### Which loop?
A good rule of thumb for parallelizing loops is start with the outer loop first. We can easily keep the code structure the exact same and simply drop in a single [par_iter_mut](https://docs.rs/rayon/latest/rayon/iter/trait.IntoParallelRefMutIterator.html#tymethod.par_iter_mut) on the outer loop:
```rust
C.par_iter_mut().enumerate().for_each(|(i, row)| {
	for k in 0..N {
		for j in 0..N {
			row += A[i][k] * B[k][j]
		}
	}
}
```
After running:
```
Time taken: 6.75999719s
```
we see a fantastic 4x speedup! Applying the same technique to the k-loop:
```rust
C.par_iter_mut().enumerate().for_each(|(i, row)| {
	row.par_iter_mut().enumerate().for_each(|(k, val)|
		for j in 0..N {
			*val += A[i][k] * B[k][j]
		}
	}
}
```
we get a significant 33% speedup: `Time taken: 4.749579936s`!
As as sanity check, I confirmed that all 16 of my cores were being utilized though monitoring the core usage with `htop`.

### What's the speed limit?
Thus far, we've blindly tried optimization techniques with no sense of what an optimal solution really is. Time for a bit of math. Consider the following specs for my machine and let's calculate it's maximum possible number of FLOPS:
```
| Feature                  | Specification |
| ------------------------ | ------------- |
| Clock frequency          | 2.9GHz        |
| Processors               | 2             |
| Cores / Processor        | 8             |
| Float instrs. / Cycle    | 16            |

Max flops = (3.3 x 10^9) x 2 x 8 x 16 = 845 GFLOPS
```

Now that we know what the hard limit of our machine, how far away from the limit are we?
```
2n^3 = 2(2^12)^3 = 2^37 floating-point operations
Running time = 4.75s
=> Flops = 2^37 / 4.75 = 29 GFLOPS
=> Utilization = Flops / Max flops = 29 / 845 = 3.4%
```
We've only achieved 3.4% of the maximum😕. That really makes you appreciate the engineering work behind hyper-optimized libraries such as `numpy`:
```python
import random
import numpy as np
from time import *

N = 4096

A = np.array([[random.random() for _ in range(N)] for _ in range(N)])
B = np.array([[random.random() for _ in range(N)] for _ in range(N)])

start = time()
C = np.matmul(A, B)
end = time()

print(f"Time taken: {end - start}")
```
```
Time taken: 0.5340592861175537
```

### What's next?
Well, we clearly have a lot to improve on. To name just a few directions we could travel down for further speedups:
- [tiling](https://learn.microsoft.com/en-us/cpp/parallel/amp/walkthrough-matrix-multiplication?view=msvc-170#multiplication-with-tiling)
- [parallel divide-and-conquer](https://en.wikipedia.org/wiki/Matrix_multiplication_algorithm#Divide-and-conquer_algorithm)
- maximizing [automatic vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization)
- [AVX intrinsics](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)

Each of these adventures could be their own blog posts, so for the sake of brevity lets call it a day here, and feel happy about the speedups we were able to achieve.

[Full code on GitHub](https://github.com/CadenMG/matrix-optimizations)
