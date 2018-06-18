---
layout: post
title:  "Does your multiprocessor really need a CAS?"
date:   2015-5-31
---

<style>
.tablelines table, .tablelines td, .tablelines th {
        border: 1px solid black;
        }
</style>


| First Header  | Second Header |
| ------------- | ------------- |
| Content Cell  | Content Cell  |
| Content Cell  | Content Cell  |

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

Since the whole instruction is performed atomically, `CAS` is useful when it comes to implement multi-threaded applications and we need the threads to synchronize in some way. For example, most non-blocking algorithm mplementations use `CAS`, from [linked lists](https://timharris.uk/papers/2001-disc.pdf) to [binary search trees](https://dl.acm.org/citation.cfm?id=2555256).

Most likely, you have heard or even used `CAS` at some point in your life.
You might be using `CAS` even without knowing it, by for instance, using a [`ConcurrentLinkedQueue`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedQueue.html) in Java.
You might also have heard that `CAS` is necessary to implement any non-blocking (e.g., lock-free) algorithm. It is for this reason that most modern processors support `CAS`. But is this really the case? Surprisingly the answer is **no**. 

Maybe during your university years, you have taken a course in concurrency and learned about consensus, consensus number, consensus (or Herlihy's) hierarchy, etc., so you might think that "We definitely need a synchronization primitive with a consensus number of infinity such as `CAS`, otherwise we will not be able to implement non-blocking algorithms!" At least, this is what I would have said until recently. After all, the "selling point" of the consensus hierarchy is to decide which instructions to put in a processor based on their synchronization power.

However, this is not the case: We can implement any non-blocking algorithm for any number of threads using synchronization primitives that have consensus number one. Yes, one! In this post, you see why the consensus hierarchy does not apply in actual systems. 

Do not worry if all the previous terminology confused you. I will explain what non-blocking algorithms are, the consensus problem and the consensus hierarchy, the relation between `CAS` and the consensus problem, and why the consensus hierarchy does not apply in practice. At the end you will understand why *modern multiprocessors do not really need the `CAS` instruction*.


## Non-blocking Algorithms
Let us say that we want to implement the simplest possible concurrent (i.e., thread-safe) algorithm, namely a concurrent counter. 
You could easily implement such a counter by first implementing a sequential (i.e., non thread-safe) counter. Then, to make it concurrent, you can introduce a global lock. Each of your sequential functions would first need to `lock` this global lock, and at the end `unlock` it, as seen below for the `increment` function.

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

The approach of using a global lock to transform a sequential algorithm to a concurrent one has the advantages that it is easy to implement such an algorithm and verify it is correct.
Howerver it has some notable drawbacks.
One issue with such an implementation is that it does not scale: as we increase the number of threads, threads will contend more on this global lock. 
Another issue related to performance is the following: What if a thread that holds the lock suddenly stops executing? 
In such a case, other threads will not be able to perform any operations either since the lock is held by a stopped thread.
Such a scenario is quite likely to occur in practice. For instance, a thread might stop executing for a period of time if it gets preempted by the OS, or the thread could slow down due to a page fault, a cache miss, etc. Even worse the thread might simply crash while holding the lock. To be frank if a thread crashes, the whole process will crash as well. But it could be that instead of threads we have processes that are sharing memory.
In such a scenario, it could be that one process fails while holding the lock, while all the other processes are stuck forever waiting for the lock to be freed, even though this will never happen. For this reason, algorithms that use locks are also called _blocking_ algorithms since the delay of one thread has the potential of impeding the progress of other threads. Naturally, any algorithm that uses locks in some way will be a blocking algorithm.


### Wait- and Lock-freedom
Algorithms that a delay of one thread cannot impede the progress of other threads are called _non-blocking algorithms_. 
Such algorithms have the nice property that even if a thread crashes or slows-down, other threads can progress (i.e., do not block).
Since, we cannot use locks to implement non-blocking algorithms what can we use instead? 
We can use synchronization primitives such as compare-and-swap, fetch-and-add, etc.
For instance, to implement a non-blocking concurrent counter, we can simply use fetch-and-add as follows:

{% highlight c %}
int increment(int* counter) {
    int value = fetch_and_add(counter);
    return value;
}
{% endhighlight %}

This concurrent counter does not have the drawback of the previous global-lock approach. In case a thread that is about to increment the counter gets preempted for a great amount of time, other threads can keep using and incrementing this counter. Unfortunately, non-blocking algorithms are much harder to devise and verify correct. For example, the first [non-blocking binary search tree](https://dl.acm.org/citation.cfm?id=1835736) is a quite involved algorithm that was devised pretty recently (in 2010).

There are actually at least two categories of non-blocking algorithms: *lock-free* and *wait-free* ones. Specifically, a function is *wait-free* if it guarantees that if one threads keeps taking steps (i.e., the scheduler gives it time slices), then the function call will terminate in a finite number of steps. For instance, the above counter is a wait-free counter, since we are guaranteed that every call to `increment` will finish in a finite number of steps. 
A function is *lock-freed* on the other hand, if at least one thread always completes its execution in a finite number of steps. Lock-freedom allows individual threads to starve but the system as a whole makes progress, since some thread will always complete the execution of such a function. For instance, let us assume we wanted to implement the aforementioned concurrent counter using compare-and-swap. An approach would be the following:

{% highlight c %}
int increment(int* counter) {
    int value = *counter;
    while (!CAS(counter, value, value + 1)) {
        value = *counter;
    }
    return value;
}
{% endhighlight %}

The above algorithm is non-blocking but is not wait-free. To see this, think of two threads (t0 and t1) that execute as follows. I assume that the `counter` contains `0` initially and in the following table, time moves from top to bottom.

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |


| thread 0      |   thread 1    |  description |
| ------------- |:-------------:|--------:|
| value = 0     |     value = 0 |   |
|     no step   |    CAS(counter, 0, 1)| CAS succeeds |
|   no step     |     return 0  | |
|  CAS(counter, 0, 1) | value = 1 | CAS fails |
|   value = 1    |     no step |  |
|     no step   |    CAS(counter, 1, 2)| CAS succeeds |
|   no step     |     return 2  | | 
|  CAS(counter, 1, 2) | no step | CAS fails |
|  ... | ... | |


that one thread is faster than all the others, so its `CAS` always succeeds while all the other threads fail to succeed. The algorithm is lock-free instead, some function will always terminate in a finite number of steps.
Of course, if each operation is executed at most once, then `lock-freedom` is equivalent to `wait-freedom`.


## Consensus
Consensus is a problem that asks for a set of threads to agree on a common value. Specifically, each thread proposes one value and all the threads have to agree on the exact same value that was proposed by some thread.
Consensus is actually a fundamental problem in computer science that first appeared in a distributed setting in which processes try to agree on a common value over a network. 

Specifically, the consensus algorithm implements a `int propose(int value)` function that each thread calls exactly once, and the end all the threads needs to return the exact same value that corresponds to one of the given values (i.e., at least some thread should have proposed that value).

Devising a blocking-algorithm to solve consensus is fairly straight-forward. For instance:
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

Note that a lock can be implemented only using read and write instructions (Bakery's algorithm), so we can solve consensus in a blocking setting just by using read and write instructions.

Let us devise a non-blocking algorithm to solve consenus using compare-and-swap:
{% highlight c %}
int decision = -1; 

int propose(int value) {
    while (!CAS(&decision, -1, value));
    return decision;
}
{% endhighlight %}

This was straight-forward as well. Note that the above algorithms, can be executed by any number of threads and they will all return the exact same decided value. Let us see if we can devise a non-blocking algorithm to solve consensus by only using `fetch-and-add`. A solution for two threads
would be the following:
{% highlight c %}
int decision = -1;


{% endhighlight %}


But wait what if we have three threads.

For example, assume we have two threads, how could we solve consensus in such a setting. We use fetch&add. ...

If we just use read/write instructions ...

Surprisingly, fetch&add cannot be used to solve consensus among three threads. Try it! If you are interested you can find a proof here but it is beyond the discussion of this post.

On the other hand, `CAS` can be used to solve consensus for any number of threads. How would this go?


### Universal Construction
Consensus is fundamental, since if we solve consensus for any number of threads we can actually take any sequential algorithm and transform it to a wait-free concurrent one (something known as universal construction). How can this be done
any sequential object can be implemented 


### Consensus Hierarchy
Herlihy was the first to notice that different object can solve consensus for different number of threads and proposed a hierarchy that captures the power of these primitives.

This lead processor vendors to start introducing powerful primitives such as CAS in their processors. At least that's what some people believe (Vassos hadzilacos paper + impossibility results for distributed computing ...)

## In Practice 
If you asked me until recently, do we need `CAS` in multiprocessors I would say yes. But is this really the case?
In the distinguished PODC 16 a paper was presented with the title " ." The authors mention that 

"  "

But how is this possible. Herlihy's hierarchy talks about operations by themselves, but in reality in multi-processors we can apply more than one operation to a specific memory location. We can for example apply a `fetch&add`, a `xor`, etc.


[Wait-Free Synchronization](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf)
> We have tried to suggest here that the resulting theory has a rich structure, yielding a number of unexpected results with consequences for algorithm design, multiprocessor architectures, and real-time systems.


[A Quarter-Century of Wait-Free Synchronization](https://dl.acm.org/citation.cfm?id=2789164
)
> *Impact 6: Design of multiprocessors?*
> I put a question mark for this impact, because here I am speculating: I do not really know why, in the late 1980s and early 1990s, multiprocessor architects abandoned operations with low consensus number in favour of universal ones. But the timing is such that I wouldn’t be surprised to learn that these architects were influenced, at least in part, by Herlihy’s discovery that, from the perspective of wait-free synchronisation, much more is possible with operations such as compare-and-swap or load-linked/store-conditional than with operations such as test-and-set or fetch-and-add.

[Impossibility Results for Distributed Computing](https://www.morganclaypool.com/doi/abs/10.2200/S00551ED1V01Y201311DCT012)
> For example, the unsolvability of consensus in asynchronous shared memory systems where processes communicate through registers has led to manufacturers including more powerful primitives such as compare&swap into their architectures

### Compare-and-swap

You might even known that `CAS` can be used to implement any wait-free algorithm for any number of threads


**Solving consensus with smaller objects ... PODC 16, DISC17, PODC18**

## Conclusion
Hopefully, I managed to convince you that the compare-and-swap instruction is not technically needed in multiprocessors. You could still implement a wait-free algorithm using much less powerful synchronizaton primitives. However, having a single instruction that is so powerful is something extremely useful.



# Notes 

Note, that a more performant [approach](https://people.csail.mit.edu/shanir/publications/Lazy_Concurrent.pdf) would be to use multiple locks in a more fine-grained approach. We could for instance have one lock for each node of the list and just lock nodes in the region where we are about to perform a modification. Such an approach would perform better but would also substantially complicate the implementation. Still, such an approach would be blocking. 