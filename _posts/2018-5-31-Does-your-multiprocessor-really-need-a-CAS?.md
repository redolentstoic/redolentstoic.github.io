---
layout: post
title:  "Does your multiprocessor really need a CAS?"
date:   2015-5-31
---

### TL;DR: Not really but it is useful nevertheless. 

Most modern processors support the compare-and-swap (`CAS`) instruction (e.g., `CMPXCHG` in x86 architectures). 
You can think that a `CAS` instruction executes the following code **atomically**.

{% highlight c %}
boolean CAS(int *p, int v1, int v2) {
	if (*p == v1) {
        *p = v2;
        return true;    
    }
    return false;
}
{% endhighlight %}

`CAS` takes as input three arguments: one pointer and two values `v1` and v2`. It dereferences the pointer and checks if it contains value `v1` and if this is the case it stores `v2` to where the pointer points and returns `true`. Otherwise, `CAS` just returns `false`. The whole instruction is performed atomically. Hence, `CAS` is useful when it comes to implement multi-threaded applications and we need the threads to synchronize in some way.

Most likely, you have heard or even used the compare-and-swap synchronization primitive already. You might also have heard that `CAS` is necessary to implement any non-blocking (e.g., lock-free) algorithm. It is for this reason that most modern processors support `CAS`. But is this really the case? Surprisingly the answer is **no**. 

Maybe during college, you have have taken a course in concurrency and learned about consensus, consensus number, Herlihy's hierarchy, etc., so you might think that "We definitely need a synchronization primitive with a consensus number of infinity such as `CAS`, otherwise we will not be able to implement non-blocking algorithms!" Actually, this is not the case: We can implement any non-blocking algorithm for any number of threads using synchronization primitives that have consensus number one. We do so by "circumventing" Herlihy's hierarchy. 


If you got confused with all these terms, do not worry. In this post, we will first explain some basic concepts. Specifically, we weill talk about non-blocking algorithms, the consensus problem and its connection to Herlihy's hierarchy, the relation between `CAS` and the consensus problem, and why Herlihy's hierarchy does not apply in practice and hence `CAS` is not really needed.


## Non-blocking Algorithms


## Consensus

## Herlihy's Hierarchy

### Compare-and-swap

You might even known that `CAS` can be used to implement any wait-free algorithm for any number of threads

Your processor most likely has a compare-and-swap (CAS) instruction that can be used for synchronization when programming a multi-threaded application. 
