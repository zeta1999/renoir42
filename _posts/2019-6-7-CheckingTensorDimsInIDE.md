---
layout: post
mathjax: true
title: Incomplete Post, Some ways to check tensor size in IDE. post 1: python
---

If you ever try to play around with one of the tensorflow, pytorch/libtorch you will without any doubt find tracking tensor
dimensions a pain in the... neck.

It happens that with a little typing a certain number of checks can on principle be automated.

I expose in this post a proof of concept for python/mypy using type annotations.

In future posts I intend to show some similar examples with C++ & Ocaml, the methodology applied for Ocaml being on principle 
applicable given HList-like types (lists of hetereogeneous typed elements). Please note, 
these do not require 'macro metaprogramming' per se - ok, C++ templates but they are used in a straightforward way. 
So, the interesting point of the proposed approach is to limit macro/multi-stage compilation usage as much as possible, 
thus making future debugging... possible/endurable! ;-)

Today, let us focus on the [Python case](https://github.com/zeta1999/TensorDimCheckIDEPython).

The trick of the matter is to encode dimensions as a abstract type parameters. Writing code in such a generic way will allow 
later to add an extra optimization loop for metaparameter tuning: get a model which is as big as needed but not much more, 
hence allowing for faster inference and re-training.

Let us first define (abstract) dimensions: A,B,C,D.

{% highlight python %}
class A:
    pass
class B:
    pass
class C:
    pass
class D:
    pass
{% endhighlight %}

[TBC]


