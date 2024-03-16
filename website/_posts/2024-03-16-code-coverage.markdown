---
layout: post
title:  "Code coverage. You keep using that word. I do not think it means what you think it means"
date:   2024-03-16 10:21:52 +0000
categories: Testing
---

Most companies use the "code coverage" metric to have some indication of how well their code has been tested.
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

This test will have 100% code coverage of AddOne, but due to the missing expectation, it will not test anything.

Hence, code coverage is not a metric for how much of your code has been tested.

This example is a bit extreme, but in a large legacy code base situations like this can easily occur.

# 100% code coverage does not ensure well tested code

Let's look at a different example:

{% highlight cpp %}
int AddOrSubtract(int in, bool subtract, int value)
{
  if (subtract)
    return in - value;
  else 
    return in + 1;
}

TEST(MyTest, TestsAbsolutelyNothing)
{
  EXPECT_EQ(2, AddOrSubtractOne(1, false, 1));
  EXPECT_EQ(0, AddOrSubtractOne(1, true, 1));
}
{% endhighlight %}

In this test, we still have 100& code coverage, but there is an obvious error in the code still.

In this case, code coverage does mean that we have tested 100% of the code, but we have still tested it poorly.

# Inconcievable!

Code coverage is a poor term. Most people have the notion that if code coverage is high, the code has been tested.
There is no direct correlation between code coverage and the amount of code that has been tested. As we saw in example 1.

Instead, we should invert the metric, and use "untested code" = 100% - code coverage. Because this is
actually the only thing we can measure in this way. It tells us how many percent of the code is
_not_ covered by any automated tests _at all_.

So, where previously your CI tool would report 80% code coverage, now should now report 20% untested code.

Now, having 0% untested code still does not mean that code has been tested well. It could still be tested poorly as in
the second example. Tests can never prove the absense of errors.

