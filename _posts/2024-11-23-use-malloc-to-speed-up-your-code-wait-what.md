---
layout: post
title:  "Use malloc to speed up your code... Wait? What?!"
date:   2024-11-23 10:21:52 +0000
categories: Performance measurements
---

# Introduction

This post comes from an experience I had while preparing a course in C++ for
internal training in my company.

As part of the course, I wanted to emphasize that one should keep variables as
automatic storage duration, and not use dynamic allocations unless necessary.

I often like to show examples of real code to prove my points, usually using
online pages like compiler explorer.

# Proving a point

I wanted to show the performance impact of using dynamic allocation vs.
automatic storage duration. Therefore, I turned to the quick-bench website
and quickly authored a very simple example:

{% highlight cpp %}
struct MyClass
{
  float f;
  int i;
  int j;
};

struct AllocatingStruct
{
  AllocatingStruct() : i(std::unique_ptr<MyClass>(new MyClass)) {}

  std::unique_ptr<MyClass> i;
};

struct NonAllocatingStruct
{
  NonAllocatingStruct() {}
  MyClass i;
};
{% endhighlight %}

The benchmark would test the time it would take to instantiate either
AllocatingStruct or NonAllocatingStruct.

{% highlight cpp %}
static void AllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    AllocatingStruct myStruct;
    benchmark::DoNotOptimize(myStruct);
  }
}

static void NonAllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    NonAllocatingStruct myStruct;
    benchmark::DoNotOptimize(myStruct);
  }
}
{% endhighlight %}

You might be wondering why I didn't use std::make_unique to make the
unique_ptr. This is because std::make_unique will zero the memory
allocated. This is also the reason we have
std::make_unique_for_overwrite, which will not zero the memory. So
if I had targeted C++20, I would probably have used
std::make_unique_for_overwrite.

Automatic allocation will not zero-initialize anything, so to make
the comparisons fair, I couldn't use std::make_unique.

I used DoNotOptimize to make sure myStruct would not be removed
by the optimizer.

Running the benchmark gave the following result:

![First benchmark](/assets/2024-11-23-use-malloc-to-speed-up-your-code-wait-what/benchmark1.png)

I was delighted to see, that the benchmark correctly proved that using dynamic
storage was way more expensive than having a variable with automatic storage
duration.

# Playing around with allocation sizes

Now, I wanted to show how this worked for different sizes of MyClass. So I
made the class a bit bigger:

{% highlight cpp %}
struct MyClass
{
  float f;
  int i[1024];
  int j;
};

{% endhighlight %}

Expecting somewhat the same results, I pressed the "Run Benchmark" button
and got...

![First benchmark](/assets/2024-11-23-use-malloc-to-speed-up-your-code-wait-what/benchmark2.png)

# Wait... What!?

In this case, the allocating class was actually _faster_ than the non-allocating.

This makes no sense. Although calls to new can be very fast if we allocate
storage that is already "cached" by our memory allocator implementation,
surely just using automatic storage must be faster. After all, it should
just be increasing a stack pointer.

So what's going on?

To answer this, we need to look at the assembly for each variant:

Non-allocating             |  Allocating
:-------------------------:|:-------------------------:
![Non-allocating assembly](/assets/2024-11-23-use-malloc-to-speed-up-your-code-wait-what/NonAllocatingStructBM.png) | ![Non-allocating assembly](/assets/2024-11-23-use-malloc-to-speed-up-your-code-wait-what/AllocatingStructBM.png)

At first, I was suspecting the compiler to optimize away the calls to new and
delete. But as can be seen from the assembly, this is not the case.

We need to dig a bit deeper into what is taking the time.

In the allocating case, the main time is spent in two mov calls, where we copy
the content of rax to the stack, and back again and then a TEST instruction.

In the non-allocating case, the main time is spent on a REP MOVSQ instruction.
In fact, we are moving 0x201 quad-words (4104 bytes) to the stack. This size
is exactly the sizeof(MyClass).

The culprit is, as you might have guessed, the call to DoNotOptimize(). Which
somehow forces the variable in question to be copied to the stack. Indeed, in
the allocating case, the contents of the unique_ptr (one pointer) is also
copied to the stack. But this is much cheaper than copying the full struct.

To make a fair comparison, we need to use the following code:

{% highlight cpp %}
static void AllocatingStructBM(benchmark::State& state) {
  for (auto _ : state) {
    AllocatingStruct myStruct;
    benchmark::DoNotOptimize(*myStruct.i);
  }
}
{% endhighlight %}

And in this case, as expected, the allocating case is slower than the
non-allocating

![Allocating slower than non-allocating](/assets/2024-11-23-use-malloc-to-speed-up-your-code-wait-what/saneresults.png)

And the assembly contains the same REP MOVSQ instruction:

![Allocating slower than non-allocating](/assets/2024-11-23-use-malloc-to-speed-up-your-code-wait-what/FixedAllocatingStructBM.png)

Sanity is restored...

# Story is not over

Now, the story is not quite over yet. I have tried to recreate the results i
found in quick-bench in [Compiler Explorer](https://godbolt.org/z/sG9fYenYd),
and on my local PC. And even with
the same compiler settings, and the same source, I cannot recreate the case
where the non-allocating case is slower than the allocating case.

Even with the same compiler version, only quick-bench insist on generating the
REP MOVSQ instructions when DoNotOptimize is used.

# Conclusion

There is not much of a conclusion other than the fact that "benchmarking is hard
to get right". I'm still puzzled as to why the compiler used by quick-bench
gives assembly that is so different from compiler explorer, with the same
compiler version. But I haven't had the time to investigate much further.