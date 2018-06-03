---
layout: post
title:  "Does your multiprocessor really need a CAS?"
date:   2015-5-31
---

### TL;DR Most likely not.

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

Since the whole instruction is performed atomically, `CAS` is useful when it comes to implement multi-threaded applications and we need the threads to synchronize in some way. For example, most non-blocking algorithm [implementations](https://github.com/LPD-EPFL/ASCYLIB) use `CAS`, from [linked lists](https://timharris.uk/papers/2001-disc.pdf) to [binary search trees](https://dl.acm.org/citation.cfm?id=2555256).

Most likely, you have heard or even used `CAS` at some point in your life.
You might be using `CAS` even without you knowing it, by for instance, using a [`ConcurrentLinkedQueue`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedQueue.html) in Java.
You might also have heard that `CAS` is necessary to implement any non-blocking (e.g., lock-free) algorithm. It is for this reason that most modern processors support `CAS`. But is this really the case? Surprisingly the answer is **no**. 

Maybe during college, you have taken a course in concurrency and learned about consensus, consensus number, Herlihy's hierarchy, etc., so you might think that "We definitely need a synchronization primitive with a consensus number of infinity such as `CAS`, otherwise we will not be able to implement non-blocking algorithms!" 
Actually, this is not the case: We can implement any non-blocking algorithm for any number of threads using synchronization primitives that have consensus number one. This is a little-known fact and initially it seems to contradict Herlihy's hierarchy. As I will show in this post, Herlihy's hierarchy but does not apply in actual systems. 

If all the previous terminology confused you, do not worry. In this post, I will explain what non-blocking algorithms are, the consensus problem and its connection to Herlihy's hierarchy, the relation between `CAS` and the consensus problem, and why Herlihy's hierarchy does not apply in practice. At the end you will understand why *modern multiprocessors do not really need the `CAS` instruction*.


## Non-blocking Algorithms
Let us say that we want to implement the simplest possible concurrent (i.e., thread-safe) algorithm, namely a concurrent counter. 
You could easily implement such a list by first implementing a sequential (i.e., non thread-safe) list. Then, you can introduce a global lock that each of your implemented functions needs to first `lock` and to `unlock` before returning, as seen below for the `i` function.

{% highlight c %}
lock* global_lock = create_lock();

int increment(int* counter) {
    int value;
    lock(global_lock);
    value = *counter;
    *counter = *counter + 1;
    unlock(global_lock);
    return value;
}
{% endhighlight %}

The idea of using a global lock to transform a sequential algorithm to a concurrent one has the advantages that it is easy to implement such an algorithm and verify correct.
Howerver it has some notable drawbacks.
One issue with such an implementation is that it does not scale: as we increase the number of threads threads will contend on the global lock. 
Another issue related to performance that occurs is the following: What if a thread that holds the lock suddenly stops executing before performing the unlock? Such a scenario is quite likely to occur in practice. For instance, a thread might stop executing for a period of time if it gets preempted by the OS, or the thread could slow down due to a page fault, a cache miss, etc. Even worse the thread might simply crash while holding the lock. To be honest if a thread crashes, the whole process will crash as well. But it could be that instead of threads we have processes that are sharing memory where the concurrent list resides. In such a scenario, it could be that one process fails while holding the lock, while all the other processes are stuck forever waiting for the lock to be freed, even though this will never happen. For this reason, algorithms that use locks are also called _blocking_ algorithms since the delay of one thread has the potential of impeding the progress of other threads. Naturally, any algorithm that uses locks in some way will be a blocking algorithm.

Note, that a more performant [approach](https://people.csail.mit.edu/shanir/publications/Lazy_Concurrent.pdf) would be to use multiple locks in a more fine-grained approach. We could for instance have one lock for each node of the list and just lock nodes in the region where we are about to perform a modification. Such an approach would perform better but would also substantially complicate the implementation. Still, such an approach would be blocking. 

### Wait-freedom
Algorithms that a delay of one thread cannot impede the progress of other threads are called _non-blocking algorithms_. 
Such algorithms have the nice property that even if a thread crashes or slows-down, other threads can keep operating.
Since, we cannot use locks to implement non-blocking algorithms what can we use instead? 
We can use synchronization primitives such as compare-and-swap, fetch-and-add, etc.
For instance, to implement a non-blocking concurrent counter, we can simply use fetch-and-add as follows:

{% highlight c %}
int increment(int* counter) {
    int value = fetch_and_add(counter);
    return value;
}
{% endhighlight %}

This concurrent counter does not have the drawback of the previous approach. In case a thread that is about to increment the counter gets preempted for a great amount of time, other threads can keep using and incrementing this counter.
Unfortunately, although not obvious from this example, non-blocking algorithms are much harder to devise and verify correct.

There are actually at least two categories of non-blocking algorithms: *lock-free* and *wait-free* ones. A *wait-free* algorithm guarantees that if one threads keeps taking steps (i.e., the scheduler provides its with quantam), then each function call will finish in a finite number of steps. For instance, the above counter is a wait-free counter implementation. *Lock-freedom* on the other hands asks that at least one thread always finish in a finite number of steps. Lock-freedom allows individual threads to starve but the system as a whole makes progress. For instance, let us assume we wanted to implement the aforementioned concurrent counter using compare-and-swap. An approach would be the following:

{% highlight c %}
int increment(int* counter) {
    int value = *counter;
    while (!CAS(counter, value, value + 1)) {
        value = *counter;
    }
    return value;
}
{% endhighlight %}

The above algorithm is also non-blocking but is lock-free. To see this, think that one thread is faster than all the others, so its `CAS` always succeeds while in the others it fails ....

Of course, if each operation is executed at most once, then `lock-freedom` is equivalent to `wait-freedom`.

For instance, how would we implement such a non-blocking list? Let us see one of the most well-known lock-free non-blocking algorithms ...


## Consensus
Consensus is a problem that asks for a set of threads to agree on a common value. Specifically, each thread proposes one value and all the threads have to agree on the exact same value that was proposed by some thread.
Consensus is actually a fundamental problem in computer science that first appeared in a distributed setting in which processes try to agree on a common value over a network. 

Specifically, the consensus algorithm implements a `int propose(int value)` function that each thread calls, and the end all the threads needs to return the exact same value that corresponds to one of the given valuess.

Devising a blocking-algorithm to solve consensus is a really easy problem. For instance:
{% highlight c %}
lock* global_lock = create_lock();
int decision = -1; 

int propose(int value) {
    lock(global_lock);
    if (decision == -1) { // has not been decided yet
        decision = value;
    }
    unlock(global_lock);
    return decision;
}
{% endhighlight %}

Let us devise a non-blocking algorithm to solve consenus using compare-and-swap:

{% highlight c %}
int decision = -1; 

int propose(int value) {
    while (!CAS(&decision, -1, value));
    return decision;
}
{% endhighlight %}


Note that the above algorithms, can be executed by any number of threads and they will all return the exact same decided value. 
Let us think if we can devise a non-blocking algorithm to solve consensus. We can try by using fetch-and-add. A solution would be:


But wait. What if we have 3 threads? 


In shared memory, consensus referes to finding a wait-free solution. For example, assume we have two threads, how could we solve consensus in such a setting. We use fetch&add. ...


Surprisingly, fetch&add cannot be used to solve consensus among three threads. Try it! If you are interested you can find a proof here but it is beyond the discussion of this post.


On the other hand, `CAS` can be used to solve consensus for any number of threads. How would this go?

### Herlihy's Hierarchy
Herlihy was the first to notice that different object can solve consensus for different number of threads and proposed a hierarchy that captures the power of these primitives.

Put a table here ...

This lead processor vendos to start introducing powerful primitives such as CAS in their processors. At least that's what some people believe (Vassos hadzilacos paper + impossibility results for distributed computing ...)

## In Practice 
If you asked me until recently, do we need `CAS` in multiprocessors I would say yes. But is this really the case?
In the distinguished PODC 16 a paper was presented with the title " ." The authors mention that 

"  "

But how is this possible. Herlihy's hierarchy talks about operations by themselves, but in reality in multi-processors we can apply more than one operation to a specific memory location. We can for example apply a `fetch&add`, a `xor`, etc.


### Universal Construction
Consensus is fundamental, since if we solve consensus for any number of threads we can actually take any sequential algorithm and transform it to a wait-free concurrent one (something known as universal construction). How can this be done
any sequential object can be implemented 


### Compare-and-swap

You might even known that `CAS` can be used to implement any wait-free algorithm for any number of threads


**Solving consensus with smaller objects ... PODC 16, DISC17, PODC18**

## Conclusion
Hopefully, I managed to convince you that the compare-and-swap instruction is not actually needed in multiprocessors. Nevertheless, this does not mean that it is far from useful and it will likely keep being used ... 
