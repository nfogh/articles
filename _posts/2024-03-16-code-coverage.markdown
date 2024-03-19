---
layout: post
title:  "Code coverage. You keep using that word. I do not think it means what you think it means"
date:   2024-03-16 10:21:52 +0000
categories: Testing
---

Most companies use the _code coverage_ metric to have some indication of how well their code has been tested.
However, as many have pointed out, code coverage is a very poor metric for indicating whether or not your code has
been tested well.

# 100% code coverage = 0% code tested

Indeed, observe the following test:

{% highlight cpp %}
int AddOne(int in)
{
  return in - 1;
}

TEST(MyTest, TestsAbsolutelyNothing)
{
  const auto val = AddOne(1);
}
{% endhighlight %}

This example is a bit extreme, but in a large legacy code base, situations like this can easily occur. This test will have 100% code coverage of AddOne, but due to the missing expectation, it will not test anything.

<div style="text-align:center; padding:20px; font-size:20pt">There is no direct correlation between <i>code coverage</i> and the amount of code that has been tested.</div>

This doesn't mean that code coverage is a useless metric. Code with a high code coverage tends to be better
tested than code with a very low code coverage metric. But relying on high code coverage can lull us into
a false sense of security.

# Inconcievable!

The problem is that _code coverage_ is a poor term. The issue stems from the term _coverage_. When you have in
your head that something is "covered", you immediately think that it has been taken care of (testing wise).
With code coverage, as we have seen, this is not the truth.

Let's look at a different example:

{% highlight cpp %}
int Add(int in, int value)
{
    return in + value;
}

int Subtract(int in, int value)
{
  return in - 1;
}

TEST(MyTest, TestAddAndSubtract)
{
  EXPECT_EQ(2, Add(1, 1));
  EXPECT_EQ(0, Subtract(1, 1));
}
{% endhighlight %}

In this test we have added expectations, and still have 100% code coverage. This will suggest that the
code is tested, but there is an obvious error in the code still.

# Inverting the term

Instead, we should invert the metric, and use another term. In lack of a better name, I will use the
term "Legacy Code". This term, I have borrowed from Michael C. Feathers excellent book "Working Effectively with
Legacy Code", wherein he describes legacy code as code with no tests. "Legacy code%" is then equal to 
"100% - code coverage%". And this metric accurately tells us how  many percent of the code has not been 
tested _at all_.

So, where previously your CI tool would report 80% code coverage, now should now report 20% legacy code.

<div style="text-align:center; padding:20px; font-size:20pt">Legacy code% = 100% - code coverage%</div>

Now, having 0% legacy code still does not mean that code has been tested well. It could still be tested 
poorly as in the second example. So the metric is still fairly flawed. In the end, tests can never prove
the absense of errors.

However, I believe that getting rid of the word "coverage" will impove peoples understanding of what this
metric actually means.
