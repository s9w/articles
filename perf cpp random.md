# Performance comparisons of C++ PRNGs
C++11 introduces a nice `random` [header][3] which contains a variety of generators and distributions. They're typically used like this:

```c++
std::default_random_engine generator;
std::uniform_int_distribution<int> distribution(0,42);
int random_number = distribution(generator);
```

Let's see how they and their boost counterparts perform on the runtime side. No evaluation is being done on the quality of the random numbers! 

## Integers
The engines are plugged into a `uniform_int_distribution<unsigned long>`. The runtime is for generating 10'000'000 numbers and writing them into a `std::vector`. As a comparison, a constant number is written into the vector, aka "[xkcd random][4]".

PRNG                   | rng.max() | runtime [ms] | runtime [arb]
---------------------- | --------- | -----------: | ---:
`rand()`               | 2^31-1    |  71.5 | 0.39
`default_random_engine`| 2^31-2    | 190.6 | 1.05
`mt19937`              | 2^32-1    | 182.3 | **1.00**
`mt19937_64`           | 2^64-1    | 181.3 | 0.99
`minstd_rand`          | 2^31-2    | 164.6 | 0.90
boost `mt19937`        | 2^32-1    | 160.1 | 0.88
boost `mt19937_64`     | 2^32-1    | 262.1 | 1.44
constant -> int           | -      |   3.8 | 0.02
constant -> uint_fast64_t | -      |   7.7 | 0.04


- Not that much of a difference between the most common engines.
- The inclusion of `rand()` is just for comparison, keep in mind that it is not thread safe, usually has horrible numeric characteristics, and commonly (at least under mingw) only has a 15bit range.
- Be aware that these runtimes change when plugged into an inappropriate data type. `mt19937_64` needs an `uint_fast64_t` to contain the full range.
- The overhead for handling the vector is relatively small, 2.1% and 4,3% of the runtime of the mt and mt62 calls respectively. So the random number generation is dominating this comparison.
- Boost is slightly faster on the 32bit and a lot slower on the 64bit side. Runtimes of other boost generators can be estimated from a table in the [documentation][5].

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
`default_random_engine` | float | 42.7 | 0.95
`mt19937 rng_mt` | float | 44.8 | **1.00**
`mt19937_64` | float | 85.2 | 1.90
`minstd_rand` | float | 42.6 | 0.95
`default_random_engine` | double | 97.9 | 2.19
`mt19937` | double | 88.7 | 1.98
`mt19937_64` | double | 88.1 | 1.97
`minstd_rand` | double | 98.0 | 2.19

No surprises here, choose what you want.

A little warning: I encountered an odd performance drop (around a factor of 10!) with the specific combination of clang compiler, `mt19937` (32 and 64bit) and `uniform_real_distribution` (float and double). But I could only reproduce this with clang (3.4), and that particular combination. Do a little testing beforehand if this applies to you. [Link][1] to stackoverflow discussion.

## Versions, source code
- g++ 4.8.2 on 64bit linux
- boost 1.53

Source code [here][2]


  [1]: http://stackoverflow.com/questions/23240586
  [2]: https://github.com/s9w/perf_cpp_random
  [3]: http://www.cplusplus.com/reference/random/
  [4]: http://xkcd.com/221/
  [5]: http://www.boost.org/doc/libs/1_55_0/doc/html/boost_random/reference.html#boost_random.reference.generators
