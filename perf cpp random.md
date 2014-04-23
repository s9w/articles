# Performance comparisons of C++ PRNGs
## Integers
Some of C++11's new pseudo-random number engines, plugged into a `uniform_int_distribution<unsigned long>`. The runtime is for generating 10000000 numbers and writing them into a `std::vector`.

PRNG                  | rng.max()  | runtime [ms]
--------------------- | ---------- | -----------:
`rand()`                | 2^31-1 |  71.5
`default_random_engine` | 2^31-2  | 190.6
`mt19937`               | 2^32-1 | 182.3
`mt19937_64`            | 2^64-1 | 181.3
`minstd_rand`           | 2^31-2 | 164.6


- Not that much of a difference between the most common engines.
- The inclusion of rand() is just for comparison, keep in mind that it is not thread safe, usually has horrible numeric characteristics, and commonly (at least under mingw) only has a 15bit range.
- Be aware that these times change when plugged into an int. But we're looking at a 64bit architecture, so we need `unsigned long`'s.

## Random bit generation
A common requirement for random data is random bits, for example for spins in physics. A naive approach for generating them would be a `dist_int(0,1)` or simply `rng()%2`. But that makes an expensive RNG call every time when we only need one of the 32 or 64 bits.

A more efficient way is generating a number, and using one of its bits as long as there are unused ones. A glance at the table above reveals that we get the most random bits per runtime from the `mt19937_64` generator. An implementation of this idea could be:

```c++
// static if recalled in a function
static unsigned long random_ulong = rng();
static int shifts = 0;
if( shifts >= 63 ){
	random_ulong = rng();
	shifts = 0;
}
random_bit = (random_ulong >> shifts) & 1;
shifts++;
```
Comparing the two in a scenario where 10000000 random bits are written in a sequential `vector`:

. | runtime [ms] | runtime [arb]
--- | ---: | ---:
`rng()%2` | 39.3 | 4.16
shifting | 9.4 | **1.00**

So it's a huge boost for a little trick.

One pitfall: Be aware of your RNGs number range. If the range is something like $2^{31}-2$ like the `default_random_engine` in my test, there is one number missing. Specifically the "11111....111" configuration. The error introduced by that is 1/(2^31-1) in this case. This is a small error, but keep this in mind for very numerically sensitive tasks. This applies to the shifting method as well as to the `rng()%2` way. A proper `dist_int(0,1)` should fix this though.

## Floats and doubles

The PRNGs are plugged into float and double `uniform_real_distribution` and measured filling a 10000000 sized `vector`:

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

A little warning: I encountered an odd performance drop (around a factor of 10!) with the specific combination of clang compiler, `mt19937` (32 and 64bit) and `uniform_real_distribution` (float and double). But I could only reproduce this with clang (3.4), and that particular combination. Do a little testing beforehand if this applies to you. [Link][1] to stackoverflow discussion

## Versions, source code
g++ 4.8.2 on 64bit linux


  [1]: http://stackoverflow.com/questions/23240586
