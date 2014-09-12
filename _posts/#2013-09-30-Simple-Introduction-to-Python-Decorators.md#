---
layout: post
title: Simple Introduction To Python Decorators
---

{{ page.title }}
================

Most of the information about the decorators on the web is pretty confusing.  Here is my take on it.

Let us take a function

{% highlight python %}
def add(a,b):
    return a+b
{% endhighlight %}

We call call this function using

{% highlight python %}
>> add(4,5)
{% endhighlight %}

Which gives

{% highlight python %}
>> 9
{% endhighlight %}

No if we write a function which takes this function as input

{% highlight python %}
def dec(our_func):
    def new_function(*args,**kwargs):
        Result = our_func(*args,**kwargs)
        print "The result is ",Result
    return new_function
{% endhighlight %}

We could now do this

{% highlight python %}
>> ADD = dec(add)

>> ADD(4,5)

>> The result is  9
{% endhighlight %}

We can also overwrite the original function with the new function

{% highlight python %}

>> add = dec(add)
{% endhighlight %}
now

{% highlight python %}

>> add(4,5)
{% endhighlight %}
will give

{% highlight python %}
>> The result is 9
{% endhighlight %}
We can also do this with the shortcut @ symbol in python

{% highlight python %}

@dec
def add2(a,b):
    return a+b

>> add2(4,5)

>> The result is 9
{% endhighlight %}

