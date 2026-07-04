---
title: Simple Introduction To Python Decorators
---

Most of the information about the decorators on the web is pretty confusing.  Here is my take on it.

Let us take a function

```python
def add(a,b):
    return a+b
```

We call call this function using

```python
>> add(4,5)
```

Which gives

```python
>> 9
```

No if we write a function which takes this function as input

```python
def dec(our_func):
    def new_function(*args,**kwargs):
        Result = our_func(*args,**kwargs)
        print "The result is ",Result
    return new_function
```

We could now do this

```python
>> ADD = dec(add)

>> ADD(4,5)

>> The result is  9
```

We can also overwrite the original function with the new function

```python

>> add = dec(add)
```
now

```python

>> add(4,5)
```
will give

```python
>> The result is 9
```
We can also do this with the shortcut @ symbol in python

```python

@dec
def add2(a,b):
    return a+b

>> add2(4,5)

>> The result is 9
```

The first question that pops up is if it has a practical use case. Let us take a small example.

We are interesded in measuring the time of any function. (I am aware of other methods which do not provide misleading results, but this method is used for simplicity.)


```python
import time

def add(a,b):
    start = time.clock()
    c = a + b
    stop = time.clock()
    timetaken = stop - start
    return c, timetaken

```

This will provide the time as shown below. However adding this to every function is repetetive. 

```python
>>> add(2,3)
(5, 2.000000000002e-06)
```

So we can use decorators to solve the problem easily. First we define the decorator as shown below.

```python
def timeit_dec(our_func):
    def new_function(*args,**kwargs):
    	start = time.clock()
        Result = our_func(*args,**kwargs)
        stop = time.clock()
        timetaken =  stop - start
        return Result, timetaken
    return new_function
```

Now it is very simple to write any code which has to be timed.

```python
@timeit_dec
def add2(a,b):
    return a+b
```

The result will be identical to the longer add function we wrote (The time will be different.). 

```python
>>> add2(2,3)
(5, 3.9999999999970615e-06)
```

Of course python macros are still not as powerful as LISP, but they are pretty useful.
