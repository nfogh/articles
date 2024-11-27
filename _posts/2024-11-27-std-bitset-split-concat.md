---
layout: post
title:  "split and concat for std::bitset"
date:   2024-11-27 10:21:52 +0000
categories: C++
---

When working with binary data, one often needs to split a larger
std::bitset into several smaller std::bitsets. One example is if you
have a memory-mapped register read into a std::bitset.

The reverse is also needed to concatenate sub-bitsets into a larger bitset.

Currently, splitting and concatenating these requires using mask and
shift operations on the bitset, which can be complicated to read and
thus, error-prone. I find that the usability of std::bitset is limited
as you still need to do a lot of unnecessary bit-twiddling.

I recently authored two utility functions (split and concat) for
std::bitset which does exactly this.

{% highlight cpp %}
template<size_t... slices, size_t num_bits_in_source>
constexpr auto split(std::bitset<num_bits_in_source> source)

template<size_t num_bits, size_t... num_bits_rest>
constexpr auto concat(std::bitset<num_bits> bits,
std::bitset<num_bits_rest>... rest)
{% endhighlight %}

# Isolating bits in registers

The use of these functions could be as follows:

{% highlight cpp %}
  std::bitset<8> uxlcr{"10010111"};
  auto [divisor_latch_access, break_ctrl, parity, parity_enable,
stopbit, word_length] = split<1,1,2,1,1,2>(uxlcr);
  word_length = 0b10;
  uxlcr = concat(divisor_latch_access, break_ctrl, parity,
parity_enable, stopbit, word_length);
{% endhighlight %}

# Working with base64-encoded data

Another use-case is extracting sextets to generate base-64 encoded data:

**To base64**
{% highlight cpp %}
const auto datablock = std::bitset<24>{...};
const auto sextets = std::split<6,6,6,6>(data);
const auto ascii = sextets2chars(sextets);
{% endhighlight %}

**From base64**
{% highlight cpp %}
const auto chars = std::array<char, 4>{...};
const auto sextets = chars2sextets(chars);
const auto binary = std::concat(sextets);
{% endhighlight %}

sextets will in these examples be a tuple of 4 std::bitset<6>

Just thought I would share.

# Code

The code can be seen below or in [compiler explorer](https://godbolt.org/z/P8PaqvnMa).

{% highlight cpp %}
#include <iostream>
#include <type_traits>
#include <utility>
#include <bitset>

template<size_t NumBitsLhs, size_t NumBitsRhs>
constexpr auto concat(std::bitset<NumBitsLhs> lhs, std::bitset<NumBitsRhs> rhs)
{
    std::bitset<NumBitsLhs + NumBitsRhs> res;
    for (int index = 0; index < NumBitsRhs; index++) {
        res[index] = rhs[index];
    }
    for (int index = 0; index < NumBitsLhs; index++) {
        res[index + NumBitsRhs] = lhs[index];
    }
    return res;
}

template<size_t NumBits, size_t... NumBitsRest>
constexpr auto concat(std::bitset<NumBits> bits, std::bitset<NumBitsRest>... rest)
{
    return concat(bits, concat(rest...));
}


template<size_t Offset, size_t NumBits, size_t NumBitsInSource>
constexpr auto slice(std::bitset<NumBitsInSource> source)
{
    std::bitset<NumBits> dst;
    for (int index = 0; index < NumBits; index++) {
        dst[index] = source[index + Offset];
    }
    return dst;
}

template<size_t NumBitsInSource, size_t NumBits1>
constexpr auto split2(std::bitset<NumBitsInSource> source)
{
    std::bitset<NumBits1> dst;
    for (int index = 0; index < NumBits1; index++) {
        dst[NumBits1 - index - 1] = source[NumBitsInSource - index - 1];
    }

    std::bitset<NumBitsInSource - NumBits1> dst2;
    for (int index = 0; index < NumBitsInSource - NumBits1; index++)
    {
        dst2[NumBitsInSource - NumBits1 - index - 1] = source[NumBitsInSource - NumBits1 - index - 1];
    }
    return std::make_tuple(dst, dst2);
}

template<size_t NumBitsInSource, size_t Slice, size_t... Slices>
constexpr auto split_impl(std::bitset<NumBitsInSource> source)
{
    const auto spl = split2<NumBitsInSource, Slice>(source);
    if constexpr(sizeof...(Slices) == 1) {
        return spl;
    } else {
        return std::tuple_cat(
            std::make_tuple(std::get<0>(spl)),
            split_impl<NumBitsInSource - Slice, Slices...>(std::get<1>(spl))
        );
    }
}

template<size_t... Slices, size_t NumBitsInSource>
constexpr auto split(std::bitset<NumBitsInSource> source)
{
    static_assert( (Slices + ...) == NumBitsInSource, "All slice length must add up to source bit length");
    return split_impl<NumBitsInSource, Slices...>(source);
}

int main(int, char* argv[])
{
    std::bitset<1> bs0{"1"};
    std::bitset<2> bs1{"00"};
    std::bitset<3> bs2{"101"};
    std::bitset<4> bs3{"1111"};
    std::bitset<5> bs4{"00000"};
    std::cout << "bs0 " << bs0 << "\n"; // bs0 1
    std::cout << "bs1 " << bs1 << "\n"; // bs1 00
    std::cout << "bs2 " << bs2 << "\n"; // bs2 101
    std::cout << "bs3 " << bs3 << "\n"; // bs3 1111
    std::cout << "bs4 " << bs4 << "\n"; // bs4 00000

    const auto bits = concat(bs0, bs1, bs2, bs3, bs4);
    std::cout << "concatenated: " << bits << "\n"; // concatenated: 100101111100000

    const auto [bs0o, bs1o, bs2o, bs3o, bs4o] = split<1,2,3,4,5>(bits);

    std::cout << "Re-split:\n";
    std::cout << "bs0 " << bs0o << "\n"; // bs0 1
    std::cout << "bs1 " << bs1o << "\n"; // bs1 00
    std::cout << "bs2 " << bs2o << "\n"; // bs2 101
    std::cout << "bs3 " << bs3o << "\n"; // bs3 1111
    std::cout << "bs4 " << bs4o << "\n"; // bs4 00000

    static_assert(std::get<0>(split<1,2,3>(std::bitset<6>("101001"))) == std::bitset<1>(0b1));
    static_assert(std::get<1>(split<1,2,3>(std::bitset<6>("101001"))) == std::bitset<2>(0b01));
    static_assert(std::get<2>(split<1,2,3>(std::bitset<6>("101001"))) == std::bitset<3>(0b001));
}
{% endhighlight %}
