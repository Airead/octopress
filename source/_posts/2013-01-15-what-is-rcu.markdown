---
layout: post
title: "what is rcu, Fundamentally?"
date: 2013-01-15 15:16
comments: true
categories: kernel
---

## Introduction
Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October of 2002. RCU achieves scalability improvements by allowing reads to occur concurrently with updates. In contrast with conventional locking primitives that ensure mutual exclusion among concurrent threads regardless of whether they be readers or updaters, or with reader-writer locks that allow concurrent reads but not in the presence of updates, RCU supports concurrency between a single updater and multiple readers. RCU ensures that reads are coherent by maintaining multiple versions of objects and ensuring that they are not freed up until all pre-existing read-side critical sections complete. RCU defines and uses efficient and scalable mechanisms for publishing and reading new versions of an object, and also for deferring the collection of old versions. These mechanisms distribute the work among read and update paths in such a way as to make read paths extremely fast. In some cases (non-preemptable kernels), RCU's read-side primitives have zero overhead.
<!--more-->

## Quick Quiz 1
But doesn't seqlock also permit readers and updaters to get work done concurrently?

This leads to the question "what exactly is RCU?", and perhaps also to the question "how can RCU                
possibly work?" (or, not infrequently, the assertion that RCU cannot possibly work). This document              
addresses these questions from a fundamental viewpoint; later installments look at them from usage and          
from API viewpoints. This last installment also includes a list of references.                                  

RCU is made up of three fundamental mechanisms, the first being used for insertion, the second being            
used for deletion, and the third being used to allow readers to tolerate concurrent insertions and              
deletions. These mechanisms are described in the following sections, which focus on applying RCU to             
linked lists:                                                                                                   

1. Publish-Subscribe Mechanism (for insertion)                                                                 
2. Wait For Pre-Existing RCU Readers to Complete (for deletion)                                                
3. Maintain Multiple Versions of Recently Updated Objects (for readers)                                        

These sections are followed by concluding remarks and the answers to the Quick Quizzes.                         

## Publish-Subscribe Mechanism                                                                                     

One key attribute of RCU is the ability to safely scan data, even though that data is being modified            
concurrently. To provide this ability for concurrent insertion, RCU uses what can be thought of as a            
publish-subscribe mechanism. For example, consider an initially NULL global pointer gp that is to be            
modified to point to a newly allocated and initialized data structure. The following code fragment              
(with the addition of appropriate locking) might be used for this purpose:                                      

{% codeblock lang:c %}
struct foo {                                                                                                
    int a;                                                                                                    
    int b;                                                                                                    
    int c;                                                                                                    
};                                                                                                          
struct foo *gp = NULL;                                                                                      

/* . . . */                                                                                                 

p = kmalloc(sizeof(*p), GFP_KERNEL);                                                                        
p->a = 1;                                                                                                   
p->b = 2;                                                                                                   
p->c = 3;                                                                                                   
gp = p;                                                                                                     
{% endcodeblock %}

Unfortunately, there is nothing forcing the compiler and CPU to execute the last four assignment                
statements in order. If the assignment to gp happens before the initialization of p's fields, then              
concurrent readers could see the uninitialized values. Memory barriers are required to keep things              
ordered, but memory barriers are notoriously difficult to use. We therefore encapsulate them into a             
primitive rcu_assign_pointer() that has publication semantics. The last four lines would then be as             
follows:                                                                                                        

{% codeblock lang:c %}
1 p->a = 1;                                                                                                   
2 p->b = 2;                                                                                                   
3 p->c = 3;                                                                                                   
4 rcu_assign_pointer(gp, p);                                                                                  
{% endcodeblock %}

The rcu_assign_pointer() would publish the new structure, forcing both the compiler and the CPU to              
execute the assignment to gp after the assignments to the fields referenced by p.                               

However, it is not sufficient to only enforce ordering at the updater, as the reader must enforce               
proper ordering as well. Consider for example the following code fragment:                                      

{% codeblock lang:c %}
1 p = gp;                                                                                                     
2 if (p != NULL) {                                                                                            
3   do_something_with(p->a, p->b, p->c);                                                                      
4 }                                                                                                           
{% endcodeblock %}

Although this code fragment might well seem immune to misordering, unfortunately, the DEC Alpha CPU             
and value-speculation compiler optimizations can, believe it or not, cause the values of p->a,            
p->b, and p->c to be fetched before the value of p! This is perhaps easiest to see in the case of               
value-speculation compiler optimizations, where the compiler guesses the value of p, fetches p->a, p->          
b, and p->c, then fetches the actual value of p in order to check whether its guess was correct. This           
sort of optimization is quite aggressive, perhaps insanely so, but does actually occur in the context           
of profile-driven optimization.                                                                                 

Clearly, we need to prevent this sort of skullduggery on the part of both the compiler and the CPU. The         
rcu_dereference() primitive uses whatever memory-barrier instructions and compiler directives are               
required for this purpose:                                                                                      

rcu_read_lock();                                                                                            
p = rcu_dereference(gp);                                                                                    
if (p != NULL) {                                                                                            
  do_something_with(p->a, p->b, p->c);                                                                      
}                                                                                                           
rcu_read_unlock();                                                                                          

The rcu_dereference() primitive can thus be thought of as subscribing to a given value of the specified         
pointer, guaranteeing that subsequent dereference operations will see any initialization that occurred          
before the corresponding publish (rcu_assign_pointer()) operation. The rcu_read_lock() and                      
rcu_read_unlock() calls are absolutely required: they define the extent of the RCU read-side critical           
section. Their purpose is explained in the next section, however, they never spin or block, nor do they         
prevent the list_add_rcu() from executing concurrently. In fact, in non-CONFIG_PREEMPT kernels, they            
generate absolutely no code.                                                                                    

Although rcu_assign_pointer() and rcu_dereference() can in theory be used to construct any conceivable          
RCU-protected data structure, in practice it is often better to use higher-level constructs. Therefore,         
the rcu_assign_pointer() and rcu_dereference() primitives have been embedded in special RCU variants of         
Linux's list-manipulation API. Linux has two variants of doubly linked list, the circular struct                
list_head and the linear struct hlist_head/struct hlist_node pair. The former is laid out as follows,           
where the green boxes represent the list header and the blue boxes represent the elements in the list.          

[linux list](http://lwn.net/images/ns/kernel/rcu/Linux_list.jpg)

Adapting the pointer-publish example for the linked list gives the following:                                   

via http://lwn.net/Articles/262464/
via http://www.rdrop.com/users/paulmck/RCU/whatisRCU.html
