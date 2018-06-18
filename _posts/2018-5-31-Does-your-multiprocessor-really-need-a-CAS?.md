---
layout: post
title:  "Does your multiprocessor really need a CAS?"
date:   2015-5-31
---

<style>
table{
    border-collapse: collapse;
    border-spacing: 0;
    border:2px solid #000000;
}

th{
    border:2px solid #000000;
}

td{
    border:1px solid #000000;
}
</style>

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

The above algorithm is non-blocking but is not wait-free. To see this, think of two threads (t0 and t1) that keep callling `increment` and that execute as follows. I assume that the `counter` contains `0` initially and in the following table, time moves from top to bottom.

| thread 0      |   thread 1    |  description |
| ------------- |:-------------:|--------:|
| `value = 0`  |  `value = 0` |   |
|     no step   |    `CAS(counter, 0, 1)` | `CAS` succeeds |
|   no step     |     `return 0`  | |
|  `CAS(counter, 0, 1)` | `value = 1` | `CAS` fails since counter has already been modified |
|   `value = 1`    |     no step |  |
|     no step   |    `CAS(counter, 1, 2)`| `CAS` succeeds |
|   no step     |     `return 2`  | | 
|  `CAS(counter, 1, 2)` | no step | `CAS` fails |
|  ... | ... | |

As you can see in the above execution, thread 1 always suceeds in performing the `CAS` and has completing an `increment` execution, while thread 0 never does. Nevertheless, if a thread fails in the `CAS`, this means that same other thread succeeded and hence there is progress.

As a remark, notice that if each thread executes a function at most once, then in such a setting a `lock-free` function is equivalent to being `wait-free`.


## Consensus
Consensus is a problem that asks for a set of threads to agree on a common value. Specifically, each thread proposes one value and all the threads have to agree on the exact same value that was proposed by some thread.
Consensus is actually a fundamental problem in computer science that first appeared in a distributed setting in which processes try to agree on a common value over a network. 

The consensus algorithm implements a `int propose(int value)` function that each thread calls exactly once, and at the end all the threads needs to return the exact same value that corresponds to one of the given values (i.e., at least some thread should have proposed that value).

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

Note that a lock can be implemented only using read and write instructions ([Bakery's algorithm](https://en.wikipedia.org/wiki/Lamport%27s_bakery_algorithm)), so we can solve consensus in a blocking setting just by using read and write instructions.

Blocking consensus algorithms are not that interesting, so let us devise some non-blocking algorithm to solve consenus. We will pay attention only to wait-free algorithms. 
Before we move on, note that implementing a non-blocking wait-free algorithm for consensus is fundamental. By solving consensus for any number of threads we can actually take any sequential algorithm and transform it to a concurrent wait-free concurrent one. An algorithm that can perform such a tranformation is called a __universal construction__.
The main idea behind devisiong a universal construction algorithm is using consensus as a base to order function calls to the sequential algorithm, but this is beyond the scope of this blog post, so we will not delve into it.


We start by providing a consensus algorithm that uses compare-and-swap:
{% highlight c %}
int decision = -1; 

int propose(int value) {
    while (!CAS(&decision, -1, value));
    return decision;
}
{% endhighlight %}

This was straight-forward as well. Note that the above algorithms, can be executed by any number of threads and they will all return the exact same decided value. Let us see if we can devise a non-blocking algorithm to solve consensus by only using `fetch-and-add`. A solution for two threads would be the following:

{% highlight c %}
int proposed_values[2];
int init_counter = 0;
int *counter = &init_counter;

int propose(int value) {
    proposed_values[thread_id] = value;
    int tmp = fetch_and_add(counter);
    if (tmp == 0) { // thread was first to increment the counter
        return value; 
    }
    return proposed_valued[1 - thread_id];
}
{% endhighlight %}
The solution works as follows. We assume that each thread is associated with a `thread_id`. One thread has `thread_id = 0` while the other has `thread_id = 1`. Before the threads increment the `counter`, they store the value they want to propose to their respective slot in the `proposed_values` array. The first thread that performs the `fetch_and_add` will get `0` as a return value and is considered to "win" the consensus hence it returns its own value. Otherwise, if a thread loses, it returns the other thread's proposed value.

Assume now, we want to solve consensus between three threads by only using `fetch_and_add`, `read`, and `write` instructions. Well, we *cannot*! It is impossible.
If you are interested you can find a proof here but it is beyond the discussion of this post.
Surprisingly, using `CAS` we can solve consensus for any number of threads, while by only using `fetch_and_add`` we can solve consensus for up to two threads. This lead us to a hierarchy of instructions based on their power to solve consensus, known as the _consensus hierarchy_.



### Consensus Hierarchy
[Maurice Herlihy](http://cs.brown.edu/people/mph/) was the first to notice that different object can solve consensus for different number of threads and proposed a hierarchy that captures the power of these primitives. This hierarchy is based on the notion of a _consensus number_. An instruction ins has consensus number k if it can solve consensus in a wait-free manner for up to k threads. In the following table, you can see some instructions and their corresponding consensus number. `CAS` has consensus number of infinity since it can be used to solve consensus between an arbitrary number of threads. 

| instruction   |  consensus number   | 
| ------------- |:-------------:|
|   `CAS`       |   +oo  |
|  ...           |   .... |
| `fetch-and_add` | 2 |
| `read/write`  | 1 |

Herlihy also showed that, for any k we can devise an instruction that has this consensus number.

The folklore belief is that the fact that `CAS` is more powerful in this sense compared to other instructions, lead processor designers to introduce the `CAS` in their processors. 

Actually, a "selling-point" behind the consensus hiearchy was exactly that as you can see in the excerpt below taken from the Herlihy's paper that introduced the hierarchy.

[Wait-Free Synchronization](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf)
> We have tried to suggest here that the resulting theory has a rich structure, yielding a number of unexpected results with consequences for algorithm design, multiprocessor architectures, and real-time systems.

Others, also have hinted that the fact that compare-and-swap is so powerful was the reason that it was added in  modern processors. To prove this fact, I provide two excerpts below:

[A Quarter-Century of Wait-Free Synchronization](https://dl.acm.org/citation.cfm?id=2789164
)
> I do not really know why, in the late 1980s and early 1990s, multiprocessor architects abandoned operations with low consensus number in favour of universal ones. But the timing is such that I wouldn’t be surprised to learn that these architects were influenced, at least in part, by Herlihy’s discovery that, from the perspective of wait-free synchronisation, much more is possible with operations such as compare-and-swap or load-linked/store-conditional than with operations such as test-and-set or fetch-and-add.

[Impossibility Results for Distributed Computing](https://www.morganclaypool.com/doi/abs/10.2200/S00551ED1V01Y201311DCT012)
> For example, the unsolvability of consensus in asynchronous shared memory systems where processes communicate through registers has led to manufacturers including more powerful primitives such as compare&swap into their architectures

## In Practice 
Based on what I described so far, it seems natural that `CAS` is needed in multiprocessors. But is this really the case?
At least, that is what I thought until recently. Until I stumbled upon this paper: [A Complexity-Based Hierarchy for Multiprocessor Synchronization](https://dl.acm.org/citation.cfm?id=2933113). In it the authors state:

> Objects that support only one of these instructions as an operation
have consensus number 2 and cannot be combined to
solve wait-free consensus for 3 or more processes. However,
a system that supports both instructions can solve wait-free
binary consensus for any number of processes.

Wait, what? This paper argues, that we can use objects with consensus number 2 to have an object with consensus number infinity. How is this even possible? Let us see. The authors state " ". Herlihy's hierarchy talks about operations by themselves, but in reality in multi-processors we can apply more than one operation to a specific memory location. We can for example apply a `fetch&add`, a `xor`, etc.


There were at least two more papers that worked on this result: DISC17, PODC18. DISC17 did this and that, while PODC18 did that and the other.

## Conclusion
To conclude. You do not really need compare-and-swap in your multiprocessor to implement a non-blocking algorithm for any number of threads. I hope I convinced you for that. However, having a single instruction that is so powerful as `CAS` is pretty useful. Even if we implemented `CAS` using lower-level instructions, such an implementation is likely to be substantially less performant than using `CAS`.

