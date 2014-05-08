# Performance comparisons of C++ RNGs
C++11 introduces a nice `random` [header][3] which contains a variety of generators and distributions, including the ubiquitous Mersenne Twister 19937 algorithm in 32 and 64bit versions. They're typically used like this:

```c++
std::mt19937 generator;
std::uniform_int_distribution<int> distribution(0,42);
int random_number1 = generator;
int random_number2 = distribution(generator);
```
When called without the distribution, the output is between 0 and their maximum. To evaluate the runtime penalty of the distributions, they're measured separately.
This article compares some of the C++11 generators, their boost counterparts, "/dev/urandom", Intel's hardware [RdRand][6] and good old `rand()`.  No thorough evaluation is being done on the quality of the random numbers! 

## Call times

First the pure call times are compared, without using them with the distributions. The runtime is for generating 10'000'000 ints and writing them into a `std::vector`, which is a typical use case. To make sure the RNG speed is measured and not the vector writing overhead, a constant number is written into the vector, aka "[xkcd random][4]" to serve as a baseline. Here's the data for GCC 4.8. The plot contains additional data for clang 3.4 and 

PRNG	|	rng.max()	|	runtime [ms]	|	runtime
--------- | --------------------- | -------------------: | ---------:
baseline_int	|	-	|	3.89	|	0.12
baseline_uint_fast64_t	|	-	|	7.99	|	0.26
std::default_random_engine	|	2^31-2	|	50.28	|	1.61
std::mt19937	|	2^32-1	|	31.23	|	1.00
std::mt19937_64	|	2^64-1	|	36.55	|	1.17
std::minstd_rand	|	2^31-2	|	50.86	|	1.63
boost::random::mt19937	|	2^32-1	|	19.95	|	0.64
boost::random::mt19937_64	|	2^64-1	|	31.84	|	1.02
RdRand	|	2^32-1	|	355.67	|	11.39
/dev/urandom	|	2^32-1	|	2283.07	|	73.11
rand()	|	2^31-1	|	71	|	2.27

- Most RNGs are fast, especially Boost. On my machine, writing a boost::random::mt19937 into a vector takes just a little over 6 cycles!
- Hardware numbers are disappointingly slow.
- The inclusion of `rand()` is just for comparison, keep in mind that it usually has horrible numeric characteristics and commonly (at least under mingw) only has a 15bit range.
- The overhead for handling the vector is relatively small, so the random number generation is dominating this comparison, as expected.

## Integers
The generators are plugged into a `uniform_int_distribution<>` plus comparison to pure `rng()` calls.

PRNG                   | runtime [ms] | runtime | dist(rng)/rng() | dist(rng)-rng()
---------------------- | -----------: | ------: | --------------: | ---:
std::default_random_engine	|	183.95	|	1.04	|	3.66	|	133.67
std::mt19937	|	176.41	|	1.00	|	5.65	|	145.18
std::mt19937_64	|	184.77	|	1.05	|	5.06	|	148.22
std::minstd_rand	|	183.28	|	1.04	|	3.60	|	132.42
boost::random::mt19937	|	95.66	|	0.54	|	4.79	|	75.71
boost::random::mt19937_64	|	197.64	|	1.12	|	6.21	|	165.80
RdRand	|	421.37	|	2.39	|	1.18	|	65.70
/dev/urandom	|	2321.96	|	13.16	|	1.02	|	38.89

- Surprise: There is a huge overhead from the `uniform_int_distribution`, boost and nonboost! No idea what's going on there, especially since the random_devices seem to be less afftected. Such an overhead is inexplicable to me.

## Random bit generation
A common requirement for random data is single random bits, for example for spins in physics. A naive approach for generating them would be a `dist_int(0,1)` or simply `rng()%2`. But that makes an expensive PRNG call every time when we only need one of the 32 or 64 bits.

A more efficient way is generating a number, and extract one of its bits with a bitmask. Then bitshift the number and repeat until all bits are used up. Only then do we need a new rng call. A glance at the table above reveals that we get the most random bits per runtime from the `mt19937_64` generator. An implementation of this idea could be:

```c++
// static if recalled in a function
static uint_fast64_t random_ulong = rng();
static int shifts = 0;
if( shifts >= 63 ){
	random_ulong = rng();
	shifts = 0;
}
random_bit = (random_ulong >> shifts) & 1;
shifts++;
```

Comparing the two in a scenario where 10'000'000 random bits are written in a sequential `vector`:

. | runtime [ms] | runtime [arb]
--- | ---: | ---:
`rng()%2` | 39.3 | 4.16
shifting | 9.4 | **1.00**

So it's a huge boost for a little trick.

One pitfall: Be aware of your RNGs number range. If the range is something like 2^31-2 like the `default_random_engine` in my test, there is one number missing. Specifically the "11111....111" configuration. The error introduced by that is 1/(2^31-1) in this case. This is a small error, but keep this in mind for very numerically sensitive tasks. This applies to the shifting method as well as to the `rng()%2` way. A proper `dist_int(0,1)` should (?) fix this.

## Floats and doubles

The PRNGs are plugged into float and double `uniform_real_distribution` and measured filling a 10'000'000 sized `vector`:

PRNG | type | runtime [ms] | runtime [arb]
--- | :---: | ---: | ---:
`default_random_engine` 	|	 float  	|	42.2	|	0.95
`mt19937`               	|	 float  	|	44.2	|	**1.00**
`mt19937_64`            	|	 float  	|	85.2	|	1.93
`minstd_rand`           	|	 float  	|	42.2	|	0.95
boost `mt19937`           	|	 float  	|	48.4	|	1.10
boost `mt19937_64`           	|	 float  	|	74.9	|	1.69
/dev/urandom            	|	 float  	|	2291.3	|	51.84
RdRand                  	|	 float  	|	393.6	|	8.90
`default_random_engine` 	|	 double 	|	97.3	|	2.20
`mt19937`               	|	 double 	|	87.9	|	1.99
`mt19937_64`            	|	 double 	|	87.6	|	1.98
`minstd_rand`           	|	 double 	|	97.4	|	2.20
boost `mt19937`           	|	 double 	|	108	|	2.44
boost `mt19937_64`           	|	 double 	|	74.6	|	1.69
/dev/urandom            	|	 double 	|	4577.7	|	103.57
RdRand                  	|	 double 	|	782.5	|	17.70

TODO: comparison to call times

A little warning: I encountered an odd performance drop (around a factor of 10!) with the specific combination of clang compiler, `mt19937` (32 and 64bit) and `uniform_real_distribution` (float and double). But I could only reproduce this with clang (3.4), and that particular combination. Do a little testing beforehand if this applies to you. [Link][1] to stackoverflow discussion.

## Versions, source code
- g++ 4.8.1 on 64bit linux
- ICC 14.0.2 20140120
- clang 3.3
- boost 1.53

Source code [here][2]


  [1]: http://stackoverflow.com/questions/23240586
  [2]: https://github.com/s9w/perf_cpp_random
  [3]: http://www.cplusplus.com/reference/random/
  [4]: http://xkcd.com/221/
  [5]: http://www.boost.org/doc/libs/1_55_0/doc/html/boost_random/reference.html#boost_random.reference.generators
  [6]: http://en.wikipedia.org/wiki/RdRand
