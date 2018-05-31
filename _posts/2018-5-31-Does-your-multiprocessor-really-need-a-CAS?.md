---
layout: post
title:  "Does your multiprocessor really need a CAS?"
date:   2015-5-31
---

If you have heard about the compare-and-swap (`CAS`) synchronization primitive, you may also have heard that `CAS` is necessary to implement non-blocking (e.g, lock-free) algorithms for any number of threads. It is for this reason that most modern processors suport compare-and-swap (e.g, CHMXP in Intel). But is this really the case? Surprisingly, the answer is **no**. 

You might have taken a course in concurrency during college and know about consensus, Herlihy's hierarchy, etc., so you might think "We definitely need a synchronization primitive with a consensus number of $$ \infty $$ such as `CAS`, otherwise we will not be able to implement non-blocking algorithms for any number of threads!" This is not the case. We can actually implement any non-blocking algorithm for any number of threads using primitives that have consensus number one. But this contradicts Herlihy's hierarchy you might say! Almost ...

In this post, we will explain non-blocking algorithms, consensus and its connection the Herlihy's hierarchy, why `CAS` is useful, and why it is not really needed by circumventing Herlihy's hierarchy.


You might even known that `CAS` can be used to implement any wait-free algorithm for any number of threads

Your processor most likely has a compare-and-swap (CAS) instruction that can be used for synchronization when programming a multi-threaded application. 


You can think of a CAS instruction as one that takes three arguments:

{% highlight c %}
boolean CAS(int *p, int v1, int v2) {
	if (*p == v1) {
        *p = v2;
        return true;    
    }
    return false;
}
{% endhighlight %}