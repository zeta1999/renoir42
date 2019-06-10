---
layout: post
mathjax: true
title: Some ways to check tensor size in Python IDE
---
Nota bene: this is only a *proof of concept* at this stage.

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

The trick of the matter is to encode dimensions as a abstract type parameters. Writing code in such a generic way will 
on principle, later allow to add an extra optimization loop for metaparameter tuning, 'for free' : get a model which is as big
as needed but not much more, hence allowing for faster inference and re-training. The important nuance here is that model 
instanciation would *always* work as tensor dimensions are validated automatically.

Let us first define (abstract) dimensions: A,B,C,D.

{% highlight python %}
class A:
    def __repr__(self) -> str:
        return "A"
class B:
    def __repr__(self) -> str:
        return "B"
class C:
    def __repr__(self) -> str:
        return "C"    
class D:
    def __repr__(self) -> str:
        return "D"
{% endhighlight %}

So that some tensor can be declared as being of size $ A \times B $.

{% highlight python %}
t3: Transition.Tensor2[E, float, A, B]
  = Transition.ones2(s, float, A(), B())
{% endhighlight %}

The core trick/cheat here is allow for dimensions check to happen almost at compile/lint check before actually running the 
tensor code.
In Python, one way to do it is to have check-only code in the file toplevel while code itself is in a $ def() $ block.
Here, transition is the name of the proof of concept package we are 
[building](https://github.com/zeta1999/TensorDimCheckIDEPython)...

{% highlight python %}
import Transition
[...]

ctx: Transition.Context = Transition.Context()

[...]

conv1d : Callable[[Transition.Tensor3[E,float, A,B, C]],
    Transition.Tensor3[E,float, A, B, Q]] 
    = Transition.conv1d_(Q(), 
    float, ctx, C(), B(), B(), 
    Transition._4(), 
    Transition._3(), 
    Transition._3(), 
    Transition._3(), 
    Transition._3()) 

def example01(s: Transition.Session[E]):
    [...]
    t6: Transition.Tensor3[E, float, A, B, C] 
    = Transition.ones3(s, float, A(), B(), C())
    t7: Transition.Tensor3[E, float, A, B, Q] = conv1d(t6)
{% endhighlight %}

Then, when running the source file...
![Run python](https://zeta1999.github.io/renoir42//images/TensorDimCheckIDEPython/RunPython.png)

What this output mean, is for the model to work fine, we need to make sure $ (A,B) $ contains $ B $ and 
that $ Q $ is equal to some formula involving $ C $. (I am sure this version is wrong but this is not the point
of this post). 

The second line of output, in particular, is triggered by 

{% highlight python %}
conv1d : Callable[[Transition.Tensor3[E,float, A,B, C]],
    Transition.Tensor3[E,float, A, B, Q]] 
    = Transition.conv1d_(Q(), 
    float, ctx, C(), B(), B(), 
    Transition._4(), 
    Transition._3(), 
    Transition._3(), 
    Transition._3(), 
    Transition._3()) 
{% endhighlight %}

So that $ Q $ is an alias for the dimension of a particular convolution operation output.

A production version would typically check these with an [SMT Solver](https://rise4fun.com/z3).

Eventually, this means that we can get the IDE (say, VS Code) to point out most dimension errors via standard type checking.
Nota: for some reason this seems to work with Mypy but not with pyre-check. If you have suggestions,
comments about this particular 'problem' please do not hesitate to contact me at renoir42 _at_ yahoo.com.
Code @ [Python case](https://github.com/zeta1999/TensorDimCheckIDEPython) )

![IDE](https://zeta1999.github.io/renoir42//images/TensorDimCheckIDEPython/IDE.png)

Please note, when changing a dimension in an unappropriate way from the example above...

![IDEError](https://zeta1999.github.io/renoir42//images/TensorDimCheckIDEPython/IDEError.png)

I kind of like it already! ;-)
Full IDE integration would then mean to add an automatic run of some dummy unit test to trigger SMT checks whenever necessary...
This is left as an exercise for a next time.

Code @ [Python case](https://github.com/zeta1999/TensorDimCheckIDEPython)

Next we will have a look at Ocaml (as there are [Ocaml bidings to LibTorch](https://github.com/LaurentMazare/ocaml-torch) ) and
C++.
