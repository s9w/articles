# Test computation

The program has a randomized 2D array on which a nested loop of this kind is iterated:

	:::cpp
	double result = 0;
	for(int i=1; i<N-1; ++i)
		for(int j=1; j<N-1; ++j)
			result += grid[i][j-1] + grid[i][j+1] + grid[i-1][j] + grid[i+1][j] + grid[i][j] * some_float;

So it's a nearest neighbour summation plus a float times the current element. A 2D Ising model would be an example for this kind of computation.

# Candidates

This article compares the runtime with the following approaches:

 - Pure Python 2 with the grid as a list of lists.
 - NumPy with the grid as a `ndarray`
 - Numba accelerated NumPy
 - PyPy
 - Pure C++ with a `std::vector<std::vector< int > >`
 - C++ with the popular Eigen3 template library with a `Eigen::Matrix<int, Eigen::Dynamic, Eigen::Dynamic>`

# Results and Discussion

![Alt Text](/images/perf_2D_grid_plot.png)

For a grid length of 1600:

.   | ms | relative speedup
-------|-----|-
Python | 1021.2 | 1.0
Numpy  | 34.5 | 29.6
Numba  | 33.1 | 30.9
C++    | 7 | 145.9
Eigen  | 3  | 340.4


Numpy vastly outperforms the native python implementation. But it's valuable to recognize that the speedup doesn't just come from NumPy's arrays. If just the python list is replaced with a NumPy array but with the same nested loop, its performance is even worse than python!

The NumPy performance boost comes from the use of NumPy's broadcasting. It can be used to rewrite the loop as

	:::cpp
	grid[1:-1,1:-1] = grid[2:,1:-1]  + grid[:-2,1:-1] + grid[1:-1,2:] + grid[1:-1,:-2] + grid[1:-1,1:-1] * some_float
	result = np.sum(lattice)

This exports the looping from the interpreter to C functions which explains the boost.

I tried [Numba](http://numba.pydata.org)'s JIT compiler but it only gains about 5%.

I also tried PyPy, but it didn't improve performance for this program at all. Reasons unknown.

The jump to C++ again dramatically reduces runtime with a factor of  5 compared to NumPy. And then it gets a little complicated. For further optimization, I tried Linux, PGO (profile guided optimization) and the popular Eigen library. Rough results:

 [ms] | C++ | Eigen
 -----------------|-----|-
 Windows          | 19  | 10
 Windows with PGO | 19  | 10
 Linux            | 19  | 18
 Linux with PGO   | 12  | 12

 So apparently PGO only seems to work under Linux and Eigen is only faster under Windows. Not sure what to make of this. Possibly the later gcc version under Windows (4.8.2 vs 4.8.1) might explain part of this. For the Graph, I used the 4.8.2 version. My takeaways:

 - Both Numpy and C++ provided massive improvements and are the "Winners". Especially since NumPy is very fast to write and C++ becomes more so with C++11.
 - PGO and Eigen are in the realm of "try, but don't count on it".
 - Numba and PyPy were disappointing. For this problem, there is also more to be gained from being cleverer and low level optimizations. Improving cache hits is probably a worthwile endeavour due to the random-ish access pattern. That's also supported by high cache misses of between 20% and 40%.

 [Source Code](https://github.com/s9w/perf_2D-grid)

# Notes:

- Don't forget your C++ compiler `-O3` flag, this makes an order of magnitude difference.
- Parallel optimisations weren't the topic of this comparison, a quick test with OpenMP yielded linear improvements with my 4 cores though.
- First, pure C++ outperformed Eigen by a factor of 4. The default memory layout turned out to be non-optimal. Switching rows and columns fixed this to an exactly equal performance. So apparently this default is different from usual practice, worth keeping in mind. See [Eigen: Storage orders](http://eigen.tuxfamily.org/dox/group__TopicStorageOrders.html).
- Numpy's broadcasting is trivially easy, extremely fast in allocation time and a huge boost from normal python. For a Performance Gain per Effort side, this was the clear winner. Also it's much more sophisticated that Eigens broadcasting because it supports broadcasting between two arrays and not just with a vector.
- There is probably room for more. Maybe fiddling with GCCs `__builtin_prefetch` can further increse cache 

# Used Versions
- Python 2.7.6
- PyPy 2.0.2
- GCC 4.8.2 (Windows) and 4.8.1 (Linux)
- Eigen 3.2.1