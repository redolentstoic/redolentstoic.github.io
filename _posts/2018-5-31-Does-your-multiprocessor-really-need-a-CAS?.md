---
layout: post
title:  "Does your multiprocessor really need a CAS?"
date:   2015-5-31
---

### TL;DR: Not really, but it is useful nevertheless. 

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

Since the whole instruction is performed atomically, `CAS` is useful when it comes to implement multi-threaded applications and we need the threads to synchronize in some way. For example, most non-blocking algorithm implementations use `CAS`, from [linked lists](https://timharris.uk/papers/2001-disc.pdf) to [binary search trees](https://dl.acm.org/citation.cfm?id=2555256).

Most likely, you have heard or even used `CAS` at some point in your life. You might also have heard that `CAS` is necessary to implement any non-blocking (e.g., lock-free) algorithm. It is for this reason that most modern processors support `CAS`. But is this really the case? Surprisingly the answer is **no**. 

Maybe during college, you have taken a course in concurrency and learned about consensus, consensus number, Herlihy's hierarchy, etc., so you might think that "We definitely need a synchronization primitive with a consensus number of infinity such as `CAS`, otherwise we will not be able to implement non-blocking algorithms!" 
Actually, this is not the case: We can implement any non-blocking algorithm for any number of threads using synchronization primitives that have consensus number one. This can happen since Herlihy's hierarchy does not apply in actual systems. 


Do not worry if you do not know what all these terms mean. In this post, I will explain what non-blocking algorithms are, the consensus problem and its connection to Herlihy's hierarchy, the relation between `CAS` and the consensus problem, and why Herlihy's hierarchy does not apply in practice. At the end you will understand why modern multiprocessors do not really need the `CAS` instruction.


## Non-blocking Algorithms
Assume you want to implement a concurrent (i.e., thread-safe) linked list. You could easily implement such a list by first implementing a sequential (i.e., non thread-safe) list. Then, you can introduce a global lock that each function of your list needs to first `lock` and at the end `unlock,` as seen below for the `insert` function.

{% highlight c %}
lock* global_lock = create_lock();

void insert(node* head, int value) {
    lock(global_lock);
    // do the insertion
    unlock(global_lock);	
}
{% endhighlight %}

One issue with such a linked list implementation is that it will not really scale as we increase the number of threads since threads will compete to lock this global lock. A more performant [approach](https://people.csail.mit.edu/shanir/publications/Lazy_Concurrent.pdf) would be to use multiple locks in a more fine-grained approach. We could for instance have one lock for each node of the list and just lock nodes in the region where we are about to perform a modification. Such an approach would perform better but would also substantially complicate the implementation of our list. 

A more fundamental issue with a linked list implemention that uses locks is that when a thread is executing and holds the lock no other thread t 


## Consensus
How can we solve it ... fetch& add only for 2 threads ...

any sequential object can be implemented 


## Herlihy's Hierarchy

### In practice 
Does not work ... 

### Compare-and-swap

You might even known that `CAS` can be used to implement any wait-free algorithm for any number of threads

Your processor most likely has a compare-and-swap (CAS) instruction that can be used for synchronization when programming a multi-threaded application. 


**Solving consensus with smaller objects ... PODC 16, DISC17, PODC18**
