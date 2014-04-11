# Performance comparison between Python, NumPy, C++ and friends for a typical physics 2D grid computation

## Test problem

The program has a randomized 2D array on which a nested loop of this kind is iterated:

```cpp
double result = 0;
for(int i=1; i<N-1; ++i)
	for(int j=1; j<N-1; ++j)
		result += grid[i][j-1] + grid[i][j+1] + grid[i-1][j] + grid[i+1][j] + grid[i][j] * some_float;
```

So it's a nearest neighbour summation plus a float times the current element. A 2D Ising model would be an example for this kind of computation.

## Candidates

This article compares the runtime with the following approaches:

- Pure Python 2 with the grid as a list of lists.
- NumPy with the grid as a `ndarray`
- Numba accelerated NumPy
- PyPy
- Pure C++ with a `std::vector<std::vector< int > >`
- C++ with the popular Eigen3 template library with a `Eigen::Array<int, Eigen::Dynamic, Eigen::Dynamic>`

The relevant code for the core computation:
### Python
```python
result = 0
for i in range(1,N-1):
	for j in range(1,N-1):
		result += grid[i][j-1] + grid[i][j+1] + grid[i-1][j] + grid[i+1][j] + grid[i][j] * 1.42
```
Replacing this loop with a list comprehension did not provide significant speedups!

### NumPy + Numba
```python
grid[1:-1,1:-1] = grid[2:,1:-1]  + grid[:-2,1:-1] + grid[1:-1,2:] +
                     grid[1:-1,:-2] + grid[1:-1,1:-1] * 1.42
result = np.sum(grid)
```

### C++
```python
double result = 0.0;
for(int i=1; i<N-1; ++i)
		for(int j=1; j<N-1; ++j)
			result += grid[i][j-1] + grid[i][j+1] + grid[i-1][j] + grid[i+1][j] + grid[i][j]*1.42;
```

## Results and Discussion

![Alt Text](https://raw.githubusercontent.com/s9w/perf_2D-grid/master/perf_2D_grid_plot.png)

For a grid length of 1600:

       | ms     | relative speedup
------ | ------ | ----------------
Python | 1021.2 | 1.0
Numpy  | 34.5   | 29.6
Numba  | 33.1   | 30.9
C++    | 7      | 145.9

Numpy vastly outperforms the native python implementation. But it's valuable to recognize that the speedup doesn't just come from NumPy's arrays. If just the python list is replaced with a NumPy array but with the same nested loop, its performance is even worse than python! The NumPy performance boost comes from the use of NumPy's highly efficient broadcasting.

I tried [Numba](http://numba.pydata.org)'s JIT compiler but it only gains about 5%.

I also tried PyPy, but it didn't improve performance for this program at all. Reasons unknown.

The jump to C++ again dramatically reduces runtime with a factor of 5 compared to NumPy. And then it gets a little complicated. For further optimization I tried PGO (profile guided optimization) and the popular Eigen library, but with inconsistent results. PGO did sometimes boost performance, but varying between compiler versions. Eigen did not show any improvements, but this was not expected since no eigen specific functions were used.

- Both Numpy and C++ provided massive improvements and are the "Winners". Especially since NumPy is very fast to write and C++ becomes more so with C++11.
- PGO and Eigen are in the realm of "try, but don't count on it".
- Numba and PyPy were disappointing. For this problem, there may also more to be gained from being cleverer and low level optimizations. Improving cache hits is probably a worthwile endeavour due to the random-ish access pattern. That's also supported by high cache misses of between 20% and 40%. Maybe fiddling with GCCs `__builtin_prefetch` or a more sophisticated approach work.

[Source Code](https://github.com/s9w/perf_2D-grid)

## Notes:

- Don't forget your C++ compiler `-O3` flag, this makes an order of magnitude difference.
- Parallel optimisations weren't the topic of this comparison, a quick test with OpenMP yielded linear improvements with my 4 cores though.
- With default options, pure C++ outperformed Eigen. The default memory layout turned out to be non-optimal. Switching rows and columns fixed this to an exactly equal performance. So apparently this default is different from usual practice, worth keeping in mind. See [Eigen: Storage orders](http://eigen.tuxfamily.org/dox/group__TopicStorageOrders.html).

## Used Versions
- Python 2.7.6
- PyPy 2.0.2
- GCC 4.8.2 (Windows) and 4.8.1 (Linux)
- Eigen 3.2.1
