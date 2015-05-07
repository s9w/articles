# Generating random bits in C++
A common requirement for random data is single random bits, for example for spins in physics. A naive approach for generating them would be a `dist_int(0,1)` or simply `rng()%2`. But that makes an expensive PRNG call every time when we only need one of the 32 or 64 bits.

A more efficient way is generating a number, and extract one of its bits with a bitmask. Then bitshift the number and repeat until all bits are used up. Only then do we need a new rng call. A glance at the table above reveals that we get the most random bits per runtime from the `mt19937_64` generator. An implementation of this idea could be:

```cpp
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